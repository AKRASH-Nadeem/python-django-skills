---
name: django-elasticsearch
description: >
  Apply this skill for all Elasticsearch integration work: index design and
  mappings, the elasticsearch-py client, django-elasticsearch-dsl, indexing
  strategies (real-time via signals vs batch via Celery), full-text search,
  bool query DSL, aggregations/facets, autocomplete, highlighting, geo-search,
  zero-downtime reindexing via aliases, index lifecycle management (ILM),
  and when to use Elasticsearch vs Django ORM full-text. Use when building
  search features, faceted filtering, analytics dashboards, or log aggregation.
---

# Django + Elasticsearch Skill

> FIRST: Use Context7 with ID `/websites/elastic_co_guide_en_elasticsearch_reference_8_19`
> for the latest Elasticsearch 8.x query DSL and mapping docs.
> Also resolve `elasticsearch-py` and `elasticsearch-dsl-py` via Context7
> for the Python client API.

---

## 1. Decision: ORM Full-Text vs Elasticsearch

```
Use Django ORM (PostgreSQL full-text):
✅ < 100k documents
✅ Simple search (title, description contains X)
✅ Don't need relevance scoring
✅ Don't need aggregations/facets
✅ Don't need autocomplete
✅ Want zero extra infrastructure

Use Elasticsearch:
✅ > 100k documents (performance cliff)
✅ Complex relevance scoring (BM25, custom boosting)
✅ Faceted search (filter by category + price range + rating)
✅ Autocomplete / search-as-you-type
✅ Multi-language text analysis
✅ Aggregations (analytics dashboards)
✅ Geo-distance search
✅ Log/event analytics
✅ Near-real-time search requirements

Rule: Start with ORM full-text → migrate to Elasticsearch when limits are hit.
```

---

## 2. Installation & Client Setup

```
# requirements/base.txt
elasticsearch>=8.0.0,<9.0.0      # Official Python client
elasticsearch-dsl>=8.0.0,<9.0.0  # High-level DSL (optional but recommended)
django-elasticsearch-dsl>=8.0.0  # Django integration (auto-indexing via signals)
```

```python
# apps/search/client.py
"""
Singleton Elasticsearch client.
Rules:
- One shared client per process (connection pooling built-in)
- Retry on transient errors (connection timeout, 503)
- Never expose raw ES errors to end users — map to domain exceptions
"""
from __future__ import annotations
import logging
from functools import lru_cache
from django.conf import settings
from elasticsearch import Elasticsearch, ConnectionError, NotFoundError

logger = logging.getLogger(__name__)


@lru_cache(maxsize=1)
def get_es_client() -> Elasticsearch:
    """
    Returns the singleton ES client.
    Configured via ELASTICSEARCH_DSL in settings.
    """
    return Elasticsearch(
        hosts=settings.ELASTICSEARCH_HOSTS,
        http_auth=(
            settings.ELASTICSEARCH_USERNAME,
            settings.ELASTICSEARCH_PASSWORD,
        ) if settings.ELASTICSEARCH_USERNAME else None,
        verify_certs=settings.ELASTICSEARCH_VERIFY_CERTS,
        ca_certs=settings.ELASTICSEARCH_CA_CERTS or None,
        retry_on_timeout=True,
        max_retries=3,
        request_timeout=30,
        sniff_on_start=False,     # Disable in containerized envs
    )
```

---

## 3. Settings

```python
# config/settings/base.py
ELASTICSEARCH_HOSTS = env.list("ELASTICSEARCH_HOSTS", default=["http://localhost:9200"])
ELASTICSEARCH_USERNAME = env("ELASTICSEARCH_USERNAME", default="")
ELASTICSEARCH_PASSWORD = env("ELASTICSEARCH_PASSWORD", default="")
ELASTICSEARCH_VERIFY_CERTS = env.bool("ELASTICSEARCH_VERIFY_CERTS", default=True)
ELASTICSEARCH_CA_CERTS = env("ELASTICSEARCH_CA_CERTS", default="")

# Index prefix — prevents collisions between environments
ELASTICSEARCH_INDEX_PREFIX = env("ELASTICSEARCH_INDEX_PREFIX", default="myapp")

# django-elasticsearch-dsl config
ELASTICSEARCH_DSL = {
    "default": {
        "hosts": ELASTICSEARCH_HOSTS,
    }
}

# config/settings/testing.py
# Use a test-specific prefix so CI doesn't pollute production indices
ELASTICSEARCH_INDEX_PREFIX = "test"
# Or disable auto-indexing in tests:
ELASTICSEARCH_DSL_AUTOSYNC = False
ELASTICSEARCH_DSL_PARALLEL = False
```

---

## 4. Index Design — Mappings

```python
# apps/search/indices/product_index.py
"""
Index design rules:
1. Separate index per domain object (products, orders, users)
2. Always version your index name (use alias for zero-downtime reindex)
3. Use `keyword` for filterable/sortable fields, `text` for searchable
4. Store minimal data — reference DB for full record when needed
5. Denormalize what you need to search/filter — don't JOIN in ES
"""
from __future__ import annotations
from django.conf import settings
from elasticsearch_dsl import (
    Document, Text, Keyword, Float, Integer, Boolean,
    Date, Object, Nested, InnerDoc, analyzer, Index
)

# ── Custom Analyzers ────────────────────────────────────────────────

# English analyzer with stemming (for full-text search)
english_analyzer = analyzer(
    "english_standard",
    tokenizer="standard",
    filter=["lowercase", "asciifolding", "english_stemmer"],
)

# Autocomplete: edge-n-gram for prefix matching
autocomplete_analyzer = analyzer(
    "autocomplete",
    tokenizer="standard",
    filter=["lowercase", "autocomplete_filter"],
)

# ── InnerDoc for nested data ────────────────────────────────────────

class CategoryInnerDoc(InnerDoc):
    id = Keyword()
    name = Keyword()
    slug = Keyword()


class VariantInnerDoc(InnerDoc):
    sku = Keyword()
    price = Float()
    stock = Integer()
    attributes = Object()


# ── Main Document ────────────────────────────────────────────────────

class ProductDocument(Document):
    """
    Elasticsearch document for Product model.
    Rule: Map only what you need to search, filter, sort, or display in results.
    """
    # Identifiers
    id = Keyword()                          # UUID as string — for exact lookups

    # Searchable text fields
    name = Text(
        analyzer=english_analyzer,
        fields={
            "raw": Keyword(),               # For exact match and sorting
            "autocomplete": Text(analyzer=autocomplete_analyzer),  # For prefix search
        }
    )
    description = Text(analyzer=english_analyzer)
    brand = Keyword(fields={"text": Text(analyzer=english_analyzer)})

    # Filterable / sortable
    category = Object(doc_class=CategoryInnerDoc)
    tags = Keyword(multi=True)
    price = Float()
    rating = Float()
    review_count = Integer()
    is_active = Boolean()
    is_in_stock = Boolean()

    # Date fields
    created_at = Date()
    updated_at = Date()

    # Nested (preserved field correlations — different from Object)
    variants = Nested(doc_class=VariantInnerDoc)

    class Index:
        name = f"{settings.ELASTICSEARCH_INDEX_PREFIX}_products_v1"
        settings = {
            "number_of_shards": 2,
            "number_of_replicas": 1,
            "analysis": {
                "filter": {
                    "english_stemmer": {
                        "type": "stemmer",
                        "language": "english",
                    },
                    "autocomplete_filter": {
                        "type": "edge_ngram",
                        "min_gram": 2,
                        "max_gram": 20,
                    },
                }
            },
            "max_result_window": 10000,     # Never allow deeper than this
        }

    class Django:
        model_class = None  # Set in apps/products/documents.py

    def prepare_category(self, instance: object) -> dict:
        """Denormalize category data into the document."""
        if not instance.category:
            return {}
        return {
            "id": str(instance.category.pk),
            "name": instance.category.name,
            "slug": instance.category.slug,
        }

    def prepare_variants(self, instance: object) -> list[dict]:
        return [
            {
                "sku": v.sku,
                "price": float(v.price),
                "stock": v.stock,
                "attributes": v.attributes,
            }
            for v in instance.variants.all()
        ]
```

---

## 5. Django Integration — django-elasticsearch-dsl

```python
# apps/products/documents.py
"""
Connects the ProductDocument to the Django Product model.
django-elasticsearch-dsl uses signals to auto-sync on save/delete.
"""
from django_elasticsearch_dsl import Document, fields
from django_elasticsearch_dsl.registries import registry
from apps.products.models import Product
from apps.search.indices.product_index import ProductDocument as BaseProductDoc


@registry.register_document
class ProductSearchDocument(BaseProductDoc):
    class Django:
        model = Product
        fields = ["id", "name", "description", "price", "is_active", "created_at"]
        related_models = ["Category", "ProductVariant"]  # Re-index on related save

        # IMPORTANT: Ignore auto-indexing on every save for high-write scenarios
        # Use ignore_signals=True + Celery batch indexing instead
        ignore_signals = False       # Set True for high-traffic models
        auto_refresh = False         # Don't force refresh after every index

    def get_queryset(self):
        """Optimized queryset for bulk indexing."""
        return (
            Product.objects
            .filter(is_deleted=False)
            .select_related("category", "brand")
            .prefetch_related("variants", "tags")
        )

    def get_indexing_dict(self, instance: Product) -> dict:
        """Full document dict — used when manually indexing."""
        return {
            "id": str(instance.pk),
            "name": instance.name,
            "description": instance.description or "",
            "brand": instance.brand.name if instance.brand else "",
            "price": float(instance.price),
            "rating": float(instance.rating or 0),
            "review_count": instance.review_count,
            "is_active": instance.is_active,
            "is_in_stock": instance.stock > 0,
            "tags": list(instance.tags.values_list("name", flat=True)),
            "category": self.prepare_category(instance),
            "variants": self.prepare_variants(instance),
            "created_at": instance.created_at.isoformat(),
            "updated_at": instance.updated_at.isoformat(),
        }
```

---

## 6. Search Service — The Right Abstraction Layer

```python
# apps/search/services.py
"""
All search logic lives here.
Views call search services — they never touch the ES client or DSL directly.
Rule: Map ES results back to domain objects where needed.
Rule: Always handle ES connection errors gracefully (fallback to DB search).
"""
from __future__ import annotations
import logging
from typing import Any
from dataclasses import dataclass, field
from apps.search.client import get_es_client
from apps.core.exceptions import ServiceUnavailableError

logger = logging.getLogger(__name__)


@dataclass
class SearchResult:
    total: int
    hits: list[dict[str, Any]]
    aggregations: dict[str, Any] = field(default_factory=dict)
    took_ms: int = 0
    page: int = 1
    page_size: int = 20


@dataclass
class ProductSearchParams:
    query: str = ""
    category_ids: list[str] = field(default_factory=list)
    tag_slugs: list[str] = field(default_factory=list)
    min_price: float | None = None
    max_price: float | None = None
    min_rating: float | None = None
    in_stock_only: bool = False
    sort_by: str = "_score"    # _score | price_asc | price_desc | newest | rating
    page: int = 1
    page_size: int = 20


def search_products(params: ProductSearchParams) -> SearchResult:
    """
    Full product search with facets.
    Falls back to DB if Elasticsearch is unavailable.
    """
    try:
        return _search_products_es(params)
    except Exception as exc:
        logger.error("Elasticsearch unavailable, falling back to DB", exc_info=exc)
        return _search_products_db_fallback(params)


def _search_products_es(params: ProductSearchParams) -> SearchResult:
    from django.conf import settings
    es = get_es_client()
    index = f"{settings.ELASTICSEARCH_INDEX_PREFIX}_products"  # Points to alias

    query = _build_product_query(params)
    aggs = _build_product_aggregations()
    sort = _build_sort(params.sort_by)

    from_ = (params.page - 1) * params.page_size

    response = es.search(
        index=index,
        query=query,
        aggs=aggs,
        sort=sort,
        from_=from_,
        size=params.page_size,
        highlight={
            "fields": {
                "name": {"number_of_fragments": 1},
                "description": {"number_of_fragments": 2, "fragment_size": 150},
            },
            "pre_tags": ["<mark>"],
            "post_tags": ["</mark>"],
        },
        _source=["id", "name", "price", "rating", "review_count",
                 "is_in_stock", "category", "brand"],
    )

    hits = [
        {
            **hit["_source"],
            "_score": hit["_score"],
            "_highlight": hit.get("highlight", {}),
        }
        for hit in response["hits"]["hits"]
    ]

    return SearchResult(
        total=response["hits"]["total"]["value"],
        hits=hits,
        aggregations=_format_aggregations(response.get("aggregations", {})),
        took_ms=response["took"],
        page=params.page,
        page_size=params.page_size,
    )


def _build_product_query(params: ProductSearchParams) -> dict[str, Any]:
    """Build bool query from search parameters."""
    must: list[dict] = []
    filter_clauses: list[dict] = []

    # Full-text search
    if params.query:
        must.append({
            "multi_match": {
                "query": params.query,
                "fields": [
                    "name^3",           # Boost name matches
                    "name.autocomplete^2",
                    "description",
                    "brand^1.5",
                    "tags",
                ],
                "type": "best_fields",
                "fuzziness": "AUTO",    # Tolerate typos
            }
        })
    else:
        must.append({"match_all": {}})

    # Always filter inactive products
    filter_clauses.append({"term": {"is_active": True}})

    # Stock filter
    if params.in_stock_only:
        filter_clauses.append({"term": {"is_in_stock": True}})

    # Category filter
    if params.category_ids:
        filter_clauses.append({"terms": {"category.id": params.category_ids}})

    # Tag filter
    if params.tag_slugs:
        filter_clauses.append({"terms": {"tags": params.tag_slugs}})

    # Price range
    price_range: dict[str, Any] = {}
    if params.min_price is not None:
        price_range["gte"] = params.min_price
    if params.max_price is not None:
        price_range["lte"] = params.max_price
    if price_range:
        filter_clauses.append({"range": {"price": price_range}})

    # Rating filter
    if params.min_rating is not None:
        filter_clauses.append({"range": {"rating": {"gte": params.min_rating}}})

    return {
        "bool": {
            "must": must,
            "filter": filter_clauses,
        }
    }


def _build_product_aggregations() -> dict[str, Any]:
    """Build facet aggregations for filter UI."""
    return {
        "categories": {
            "terms": {"field": "category.id", "size": 50}
        },
        "tags": {
            "terms": {"field": "tags", "size": 100}
        },
        "price_range": {
            "stats": {"field": "price"}
        },
        "rating_distribution": {
            "histogram": {"field": "rating", "interval": 1, "min_doc_count": 1}
        },
        "in_stock_count": {
            "filter": {"term": {"is_in_stock": True}}
        },
    }


def _build_sort(sort_by: str) -> list[dict[str, Any]]:
    return {
        "_score": [{"_score": {"order": "desc"}}],
        "price_asc": [{"price": {"order": "asc"}}],
        "price_desc": [{"price": {"order": "desc"}}],
        "newest": [{"created_at": {"order": "desc"}}],
        "rating": [{"rating": {"order": "desc"}}, {"review_count": {"order": "desc"}}],
    }.get(sort_by, [{"_score": {"order": "desc"}}])


def _format_aggregations(raw_aggs: dict[str, Any]) -> dict[str, Any]:
    """Format ES aggregations into a clean API-friendly structure."""
    return {
        "categories": [
            {"id": b["key"], "count": b["doc_count"]}
            for b in raw_aggs.get("categories", {}).get("buckets", [])
        ],
        "tags": [
            {"slug": b["key"], "count": b["doc_count"]}
            for b in raw_aggs.get("tags", {}).get("buckets", [])
        ],
        "price": raw_aggs.get("price_range", {}),
        "rating_histogram": [
            {"rating": int(b["key"]), "count": b["doc_count"]}
            for b in raw_aggs.get("rating_distribution", {}).get("buckets", [])
        ],
        "in_stock_count": raw_aggs.get("in_stock_count", {}).get("doc_count", 0),
    }


def _search_products_db_fallback(params: ProductSearchParams) -> SearchResult:
    """PostgreSQL fallback when ES is unavailable."""
    from apps.products.models import Product
    from django.db.models import Q
    qs = Product.objects.filter(is_active=True, is_deleted=False)
    if params.query:
        qs = qs.filter(
            Q(name__icontains=params.query) | Q(description__icontains=params.query)
        )
    total = qs.count()
    offset = (params.page - 1) * params.page_size
    results = list(qs.values("id", "name", "price", "rating")[offset:offset + params.page_size])
    return SearchResult(total=total, hits=results, page=params.page, page_size=params.page_size)
```

---

## 7. Autocomplete Endpoint

```python
# apps/search/views.py
from __future__ import annotations
from rest_framework.views import APIView
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework.permissions import AllowAny
from django.conf import settings
from apps.search.client import get_es_client


class ProductAutocompleteView(APIView):
    """
    Search-as-you-type autocomplete.
    Returns up to 8 suggestions for a partial query.
    """
    permission_classes = [AllowAny]
    throttle_scope = "autocomplete"  # settings: "autocomplete": "60/minute"

    def get(self, request: Request) -> Response:
        query = request.query_params.get("q", "").strip()
        if not query or len(query) < 2:
            return Response({"suggestions": []})

        try:
            es = get_es_client()
            index = f"{settings.ELASTICSEARCH_INDEX_PREFIX}_products"
            response = es.search(
                index=index,
                query={
                    "multi_match": {
                        "query": query,
                        "fields": ["name.autocomplete^2", "brand", "tags"],
                        "type": "bool_prefix",
                    }
                },
                _source=["id", "name", "category.name", "price"],
                size=8,
            )
            suggestions = [
                {
                    "id": hit["_source"]["id"],
                    "name": hit["_source"]["name"],
                    "category": hit["_source"].get("category", {}).get("name", ""),
                    "price": hit["_source"].get("price"),
                }
                for hit in response["hits"]["hits"]
            ]
            return Response({"suggestions": suggestions})
        except Exception as exc:
            import logging
            logging.getLogger(__name__).error("Autocomplete ES error: %s", exc)
            return Response({"suggestions": []})
```

---

## 8. Zero-Downtime Reindexing — Alias Strategy

```python
# apps/search/management/commands/reindex_products.py
"""
Zero-downtime reindexing strategy:
1. Create a NEW index with the new mapping (v2)
2. Bulk-index all data into v2
3. Atomic alias swap: alias "products" → v2
4. Delete v1

The application always searches via the ALIAS, never a direct index name.
This means no downtime and no failed searches during reindexing.
"""
from __future__ import annotations
import logging
from django.core.management.base import BaseCommand
from django.conf import settings
from apps.search.client import get_es_client
from apps.search.indices.product_index import ProductDocument

logger = logging.getLogger(__name__)


class Command(BaseCommand):
    help = "Reindex all products with zero downtime using alias swap."

    def handle(self, *args, **options) -> None:
        es = get_es_client()
        prefix = settings.ELASTICSEARCH_INDEX_PREFIX
        alias = f"{prefix}_products"
        new_index = f"{alias}_v{self._get_next_version(es, alias)}"

        self.stdout.write(f"Creating new index: {new_index}")
        self._create_index(es, new_index)

        self.stdout.write("Bulk indexing all products...")
        self._bulk_index(es, new_index)

        self.stdout.write(f"Swapping alias '{alias}' to '{new_index}'")
        self._swap_alias(es, alias, new_index)

        self.stdout.write(self.style.SUCCESS(f"Reindex complete. Alias '{alias}' → '{new_index}'"))

    def _get_next_version(self, es: object, alias: str) -> int:
        """Get current version from alias to determine next version."""
        try:
            aliases = es.indices.get_alias(name=alias)
            current = list(aliases.keys())[0]  # e.g., "myapp_products_v2"
            return int(current.split("_v")[-1]) + 1
        except Exception:
            return 1

    def _create_index(self, es: object, index_name: str) -> None:
        mapping = ProductDocument._doc_type.mapping.to_dict()
        settings_dict = ProductDocument._doc_type.index.settings
        es.indices.create(
            index=index_name,
            mappings=mapping,
            settings=settings_dict,
        )

    def _bulk_index(self, es: object, index_name: str) -> None:
        from apps.products.models import Product
        from elasticsearch.helpers import bulk

        def generate_actions():
            qs = (
                Product.objects.filter(is_active=True, is_deleted=False)
                .select_related("category", "brand")
                .prefetch_related("variants", "tags")
                .iterator(chunk_size=500)
            )
            doc = ProductDocument()
            for product in qs:
                yield {
                    "_index": index_name,
                    "_id": str(product.pk),
                    "_source": doc.get_indexing_dict(product),
                }

        success, failed = bulk(
            es,
            generate_actions(),
            chunk_size=500,
            request_timeout=60,
            raise_on_error=False,
        )
        self.stdout.write(f"Indexed: {success} | Failed: {len(failed)}")
        if failed:
            logger.error("Bulk index failures: %s", failed[:10])

    def _swap_alias(self, es: object, alias: str, new_index: str) -> None:
        """Atomic alias swap — zero downtime."""
        actions = [{"add": {"index": new_index, "alias": alias}}]

        # Remove old index from alias if it exists
        try:
            existing = es.indices.get_alias(name=alias)
            for old_index in existing.keys():
                if old_index != new_index:
                    actions.append({"remove": {"index": old_index, "alias": alias}})
        except Exception:
            pass  # Alias doesn't exist yet — first time

        es.indices.update_aliases(body={"actions": actions})
```

---

## 9. Celery-Based Async Indexing (High-Traffic Models)

```python
# apps/search/tasks.py
"""
Use Celery for indexing when:
- Model is written to many times per second
- Auto-sync via signals would create a DB-per-request bottleneck
- You want to batch updates for efficiency

Set ignore_signals=True in Document.Django to disable auto-sync.
"""
from __future__ import annotations
from celery import shared_task
import logging

logger = logging.getLogger(__name__)


@shared_task(
    bind=True,
    name="search.index_product",
    max_retries=3,
    autoretry_for=(ConnectionError,),
    retry_backoff=True,
    queue="search",
    acks_late=True,
)
def index_product(self, product_id: str) -> None:
    """Index or re-index a single product."""
    from apps.products.models import Product
    from apps.products.documents import ProductSearchDocument

    try:
        product = (
            Product.objects
            .select_related("category", "brand")
            .prefetch_related("variants", "tags")
            .get(pk=product_id)
        )
        doc = ProductSearchDocument()
        doc.update(product)
        logger.debug("Indexed product %s", product_id)
    except Product.DoesNotExist:
        logger.warning("Product not found for indexing: %s", product_id)


@shared_task(name="search.delete_product_from_index")
def delete_product_from_index(product_id: str) -> None:
    """Remove a product from the search index."""
    from apps.products.documents import ProductSearchDocument
    from elasticsearch import NotFoundError
    try:
        ProductSearchDocument().get(id=product_id).delete()
    except NotFoundError:
        pass  # Already removed — idempotent


# Trigger from service via transaction.on_commit():
# transaction.on_commit(lambda: index_product.delay(str(product.pk)))
```

---

## 10. Search API View

```python
# apps/search/views.py (add to existing)
from rest_framework.views import APIView
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework.permissions import AllowAny
from apps.search.services import search_products, ProductSearchParams


class ProductSearchView(APIView):
    permission_classes = [AllowAny]

    def get(self, request: Request) -> Response:
        params = ProductSearchParams(
            query=request.query_params.get("q", ""),
            category_ids=request.query_params.getlist("category"),
            tag_slugs=request.query_params.getlist("tag"),
            min_price=float(p) if (p := request.query_params.get("min_price")) else None,
            max_price=float(p) if (p := request.query_params.get("max_price")) else None,
            min_rating=float(r) if (r := request.query_params.get("min_rating")) else None,
            in_stock_only=request.query_params.get("in_stock") == "true",
            sort_by=request.query_params.get("sort", "_score"),
            page=int(request.query_params.get("page", 1)),
            page_size=min(int(request.query_params.get("page_size", 20)), 100),
        )
        result = search_products(params)
        return Response({
            "total": result.total,
            "page": result.page,
            "page_size": result.page_size,
            "total_pages": -(-result.total // result.page_size),
            "took_ms": result.took_ms,
            "results": result.hits,
            "facets": result.aggregations,
        })
```

---

## 11. Health Check Integration

```python
# apps/core/views.py — add to existing health_check
def health_check(request):
    checks = {}
    ...
    # Elasticsearch check
    try:
        from apps.search.client import get_es_client
        es = get_es_client()
        info = es.cluster.health(timeout="2s")
        status = info.get("status", "red")
        checks["elasticsearch"] = "ok" if status in ("green", "yellow") else "error"
        if status == "red":
            healthy = False
    except Exception:
        checks["elasticsearch"] = "error"
        # Note: ES being down is WARNING not CRITICAL — fallback to DB
        # Don't set healthy = False here unless ES is critical path
    ...
```

---

## 12. Production Checklist

```
Index Design:
✅ `keyword` for filter/sort fields, `text` for full-text
✅ Custom analyzers (stemming, autocomplete) defined in index settings
✅ Nested docs for correlated sub-objects (variants)
✅ Object docs for flat sub-objects (category)
✅ `_source` limited to display fields — not all fields
✅ Index names versioned (v1, v2) — always searched via alias

Reliability:
✅ Graceful fallback to DB search when ES is unavailable
✅ ES connection errors caught and logged — never 500 to user
✅ Retry with backoff on transient ES failures (Celery task)
✅ Idempotent index/delete operations
✅ Zero-downtime reindex via alias swap

Performance:
✅ Bulk indexing via Celery (not per-save signal for high-traffic)
✅ `transaction.on_commit()` before dispatching index task
✅ `_source` filtering in queries (don't fetch fields you don't need)
✅ `filter` context (not `query`) for non-scoring criteria (faster)
✅ Paginate with `from` + `size` (max 10k) or `search_after` for deep

Security:
✅ ES cluster not exposed to public internet
✅ Authentication enabled on ES cluster (TLS + username/password)
✅ Application has index-level permissions only (not cluster admin)
✅ Query input sanitized before passing to ES DSL
✅ No raw user input in `script` queries

Monitoring:
✅ Index lag monitoring (time from DB write to ES indexed)
✅ Search latency tracked (ES `took` ms in logs)
✅ Failed indexing tasks tracked in Celery/Flower/Sentry
✅ ES cluster health in health check endpoint
```
