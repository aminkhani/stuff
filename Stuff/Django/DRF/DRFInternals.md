```table-of-contents
title: **Table of Contents**
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```
---
## 🧭 What DRF actually is

Django REST Framework is not a second web framework bolted onto Django. It is **one Django view class** — `APIView`, a subclass of `django.views.generic.View` — that inserts a fixed pipeline in front of your handler method: wrap the request, negotiate a content type, resolve a version, authenticate, authorize, throttle, run the handler, render. Serializers, routers, permissions and throttles are all just participants in that pipeline.

Knowing the **order** of the pipeline is what separates "it works" from "I know why it broke". Almost every production DRF bug is one of three shapes: something ran later than you assumed (authentication is lazy), something ran earlier than you assumed (`queryset` is evaluated at import), or something never ran at all (`has_object_permission` on a list endpoint).

> [!tldr] TL;DR
> Permissions run **before** the handler body, but authentication is **lazy** — it fires on the first `request.user` access. `has_object_permission` only runs when you go through `get_object()`, so **queryset scoping in `get_queryset()`** is what actually prevents cross-tenant reads. `fields = "__all__"` is a future data leak. Every relation a serializer field walks is one query per row unless `get_queryset()` prefetched it. `partial=True` is the whole difference between `PUT` and `PATCH`.

---
## 🛣️ The request/response path

A DRF endpoint is `as_view()` → `dispatch()` → your method. `as_view()` returns a plain Django view function with `csrf_exempt` applied — DRF disables CSRF at the view level, and `SessionAuthentication` turns it back on for cookie-authenticated requests. That is why a session-based `POST` from a browser still needs a CSRF token while a token-based one does not.

```python
# rest_framework/views.py — dispatch(), condensed to the parts that matter
def dispatch(self, request, *args, **kwargs):
    # 1) wrap Django's HttpRequest in DRF's Request: parsers + authenticators attached
    request = self.initialize_request(request, *args, **kwargs)
    self.request = request
    try:
        # 2) negotiation, versioning, authentication wiring, permissions, throttles
        self.initial(request, *args, **kwargs)
        handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
        # 3) only now does your code run
        response = handler(request, *args, **kwargs)
    except Exception as exc:
        # 4) an APIException becomes a Response; anything else is a real 500
        response = self.handle_exception(exc)
    # 5) pick the renderer and finish the Response object
    self.response = self.finalize_response(request, response, *args, **kwargs)
    return self.response
```

| Stage | What happens | Where it bites |
| --- | --- | --- |
| `initialize_request` | builds `rest_framework.request.Request`; attaches `parser_classes`, `authentication_classes`, the negotiator and the versioning scheme | `request.data` does not exist before this, so middleware never sees it |
| `perform_content_negotiation` | picks the renderer from `Accept` | an unsatisfiable `Accept` is a `406`, raised before your code |
| `determine_version` | sets `request.version` and `request.versioning_scheme` | an unknown version is a `404`/`406` from the pipeline |
| `perform_authentication` | installs the **lazy** `request.user` property | errors from your auth class surface at the first `request.user` read |
| `check_permissions` | `has_permission` on every `permission_classes` entry | `403` (or `401`) **before** the handler body |
| `check_throttles` | every `throttle_classes` entry | `429` with `Retry-After` |
| handler | `get`/`post`/`patch`/… or the viewset action | your code |
| `finalize_response` | sets `accepted_renderer`; rendering itself happens lazily at WSGI time | a serialization error here escapes your `try` block |

### 📦 `Request` — `.data` versus `request.POST`

DRF's `Request` is a wrapper, not a subclass, and it parses the body itself:

```python
# ❌ empty for a JSON body: Django only fills POST for form content types
username = request.POST["username"]

# ✅ parser-driven: JSON, form-encoded and multipart all land here
username = request.data["username"]

# the query string is unchanged — still a QueryDict
page = request.query_params.get("page")     # == request.GET
```

- `request.data` is **lazy and cached**. The first access runs the parser matched to `Content-Type`, and there is exactly one shot at the body stream — reading `request.body` first and `request.data` second raises `RawPostDataException`.
- `JSONParser`, `FormParser` and `MultiPartParser` are the defaults. An unsupported `Content-Type` is `415 Unsupported Media Type`, raised by the parser rather than by your validation.
- For multipart, `request.data` merges files and fields; for a JSON array body it is a `list`, which is why `Serializer(data=request.data, many=True)` works unchanged.
- Middleware that touches `request.POST` — some logging and APM middleware does — consumes the stream before DRF gets it. That is the classic "my JSON body is empty, but only in production" bug.

> [!IMPORTANT]
> `request.user` is a lazy property. DRF wires it up in `perform_authentication`, but the authenticators only run on the **first attribute access**. If no permission class and no handler line touches `request.user`, a broken or expired token produces no error at all — which means `AllowAny` endpoints never validate the credentials they were sent. The mirror image is just as confusing: an exception raised inside your `authenticate()` appears to originate wherever `request.user` was first read, so the traceback points at a permission class instead of the auth class.

Two ordering facts are worth internalising. **Permissions are checked before the handler**, so a `403` never reaches your business logic — but `has_object_permission` is not part of that check, for reasons covered below. And **throttling runs after permissions**, so an unauthorized client burns no throttle budget.

---
## 🧬 Serializers — the validation pipeline

A serializer is two functions bolted to one field declaration: `to_internal_value` for input, `to_representation` for output. `ModelSerializer` adds introspection — it reads the model's fields, generates the matching serializer fields, and supplies `create()` and `update()`. That is *all* it adds, so a model field with a custom type needs an explicit serializer field. Which model field a column should have been in the first place is a separate decision, covered in [Character Types](../Models/CharacterTypes.md).

```python
serializer = OrderSerializer(data=request.data, partial=request.method == "PATCH")
serializer.is_valid(raise_exception=True)      # -> 400 with the error dict
order = serializer.save(user=request.user)     # kwargs are injected into validated_data
```

The order of operations inside `is_valid()` is fixed, and every "my validator never runs" question is answered by it:

| Step | What runs | Notes |
| --- | --- | --- |
| 1 | `run_validation(data)` | catches `ValidationError` and Django's `ValidationError`, fills `.errors` |
| 2 | `to_internal_value(data)` | per-field coercion; unknown keys are **silently dropped** |
| 3 | field `run_validation` → `validate_empty_values` → field `to_internal_value` → field `validators` | `required` / `allow_null` / `default` are decided here |
| 4 | `validate_<field_name>(self, value)` | one field, already coerced; must return the value |
| 5 | `validate(self, attrs)` | cross-field rules; the only place that sees all values at once |
| 6 | `Meta.validators` | `UniqueTogetherValidator`, `UniqueForDateValidator`, your own |

If step 3 fails for a field, that field's `validate_<field>` is skipped; and if *any* field failed, `validate()` never runs at all. So `validate()` may assume the keys it reads are present **only** when those fields are `required` and the serializer is not partial.

### 🧾 Error shapes

The shape of a `ValidationError` follows from where you raised it, and clients depend on that shape:

```python
raise ValidationError({"email": ["Already taken."]})   # -> per-field errors
raise ValidationError(["Rule X failed."])              # -> {"non_field_errors": [...]}
raise ValidationError("bad")                           # -> {"detail": ...} at view level
# inside validate_<field>: raise ValidationError("msg") -> {"<field>": ["msg"]}
```

`raise_exception=True` turns that dict into a `400` through the exception handler; without it you must check the return value of `is_valid()` yourself. `serializer.errors` is always a dict of **lists**, and the `non_field_errors` key is configurable via `NON_FIELD_ERRORS_KEY` — clients that hard-code it break the day you change it.

### ✍️ `create`, `update` and `save(**kwargs)`

```python
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ["id", "reference", "total", "status", "created_at"]
        read_only_fields = ["id", "status", "created_at"]
        extra_kwargs = {"reference": {"validators": [validate_reference_format]}}

    def create(self, validated_data):
        # validated_data already contains whatever was passed to save(**kwargs)
        return Order.objects.create(**validated_data)
```

- `save()` dispatches to `update()` when the serializer was built with an `instance`, otherwise to `create()`. `save(user=request.user)` merges those kwargs **into `validated_data`** before dispatching, and that is the correct way to set an owner: it comes from the token, never from the request body.
- A `read_only` field never reaches `validated_data`, so a client cannot set it — that is the mass-assignment defence. A `write_only` field (a password) is validated and saved but never rendered.
- `source="profile.city"` reads a dotted path on output and writes a nested dict on input; `source="*"` hands the whole object to the field, which is how a computed field becomes writable.
- `SerializerMethodField` is always read-only and calls `get_<field_name>(self, obj)`. It is the single most common hiding place for N+1 queries, because the method body looks like ordinary Python instead of a relation traversal.
- `default=` implies `required=False`. `required=False` **without** a default means the key is simply absent from `validated_data`, so `validated_data["x"]` raises `KeyError` inside `create()` — use `.get()`.
- `extra_kwargs` is how you tweak an auto-generated field without redeclaring it; redeclaring the field explicitly overrides `extra_kwargs` entirely.

> [!WARNING]
> Unknown keys in the payload are **discarded without complaint**. A client that misspells `quanity` gets a `201` and a row with the default quantity. For a public API that matters: compare `set(self.initial_data) - set(self.fields)` in `validate()` and raise on the difference.

### 🩹 `partial=True` and PATCH

`partial=True` does exactly one thing: it makes every field `required=False` for that run. The consequences are not obvious.

| Behaviour | `partial=False` (PUT) | `partial=True` (PATCH) |
| --- | --- | --- |
| Missing required field | `400` | ignored, absent from `validated_data` |
| `default=` on a field | applied when the key is absent | **not** applied — no key means no change |
| `validate(attrs)` | sees every field | sees only the submitted keys |
| Cross-field rules | safe to read both sides | must fall back to `self.instance` for the missing side |

```python
def validate(self, attrs):
    # ❌ KeyError on PATCH when only one of the two was sent
    if attrs["start"] > attrs["end"]:
        raise serializers.ValidationError("start must precede end")

    # ✅ fall back to the stored value for anything not being changed
    start = attrs.get("start", getattr(self.instance, "start", None))
    end = attrs.get("end", getattr(self.instance, "end", None))
    if start and end and start > end:
        raise serializers.ValidationError({"start": "must precede end"})
    return attrs
```

`UniqueTogetherValidator` hits the same wall from the other side: it needs all of its fields, so on a partial update it pulls the missing ones off the instance — but only if those fields are declared on the serializer. Drop one from `fields` and the constraint check quietly vanishes from validation, resurfacing as an `IntegrityError` at `save()` time instead of a clean `400`.

### 🪆 Nested serializers and `many=True`

```python
class ItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = Item
        fields = ["sku", "qty"]

class OrderSerializer(serializers.ModelSerializer):
    items = ItemSerializer(many=True)          # writable ONLY with an explicit create()

    class Meta:
        model = Order
        fields = ["id", "reference", "items"]

    def create(self, validated_data):
        items = validated_data.pop("items")            # nested data is not a model kwarg
        order = Order.objects.create(**validated_data)
        Item.objects.bulk_create([Item(order=order, **i) for i in items])
        return order
```

Without that `create()`, `ModelSerializer.create()` raises — it refuses to guess whether nested items should be created, replaced or merged, and it is right to refuse. Nested **updates** are worse: there is no correct default for "the client sent three items and the row has five", so you have to decide (replace all, upsert by key, or reject) and write it. Wrap it in a transaction, because a partial write across two tables is exactly the failure you cannot reconcile afterwards.

`many=True` does not produce a list-aware version of your class. `__new__` returns a `ListSerializer` wrapping a child instance, and that indirection explains several behaviours:

| Aspect | Consequence |
| --- | --- |
| `validate()` | runs on the **child**, once per item. Cross-item rules need a custom `ListSerializer.validate` |
| `ListSerializer.create` | exists — it loops and calls the child's `create()` per item |
| `ListSerializer.update` | **does not exist**. `save()` on an instance-bound `many=True` serializer raises `NotImplementedError`; bulk update needs your own subclass keyed on an id |
| `list_serializer_class` | the `Meta` hook for supplying that subclass |
| Errors | a **list** of dicts positionally aligned with the input, with `{}` for the items that passed |

> [!NOTE]
> For read-only nesting, `depth = 1` in `Meta` looks tempting and is almost always wrong: it generates nested representations of *every* relation with no field control, re-introducing `__all__` one level down. Declare the nested serializer.

---
## 🐌 The N+1 trap lives in your serializer

A serializer field that walks a relation is a query executed once **per row**. Nothing signals it, because it looks like attribute access — it *is* attribute access.

```python
class OrderSerializer(serializers.ModelSerializer):
    customer_name = serializers.CharField(source="customer.name")    # 1 query per row
    line_count = serializers.SerializerMethodField()

    def get_line_count(self, obj):
        return obj.items.count()                                     # 1 more per row

    class Meta:
        model = Order
        fields = ["id", "customer_name", "line_count", "tags"]        # tags M2M: 1 more
```

With `PAGE_SIZE = 50` that page costs 1 + 50 + 50 + 50 = **151 queries**. The fix is not in the serializer, it is in the view:

```python
class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = OrderSerializer

    def get_queryset(self):
        return (
            Order.objects.filter(organization=self.request.user.organization)
            .select_related("customer")                   # JOIN — forward FK
            .prefetch_related("tags")                     # 2nd query — M2M
            .annotate(line_count=Count("items"))          # aggregate in SQL
        )
```

```python
    # and the serializer reads the annotation instead of counting per row
    line_count = serializers.IntegerField(read_only=True)
```

Three queries, whatever the page size. The rules that keep it that way:

- `select_related` for forward `ForeignKey`/`OneToOneField`, `prefetch_related` for reverse FK and `ManyToManyField`. Getting that backwards raises, so the mistake is loud — unlike forgetting entirely, which is silent and only shows up as latency.
- **`.count()` or `.filter()` inside a `SerializerMethodField` defeats `prefetch_related`**: a new filter is a new queryset, and a new queryset is a new query. Use `Prefetch(..., queryset=..., to_attr=...)` and read the attribute, or push the work into `annotate`.
- One `Count` in SQL beats fifty round trips. The ORM-side mechanics of all of this — `Prefetch`, `Subquery`/`Exists`, `only()` and the deferred-field re-query trap — are in [ORM Optimization](../ORMOptimization.md).
- Assert the query count in tests. It is the only thing that keeps this fixed after the next serializer field lands.

---
## 🧱 Views — `APIView`, generics, viewsets

| Layer | You write | Use when |
| --- | --- | --- |
| `APIView` | every method by hand | non-CRUD endpoints: a webhook receiver, a login, an RPC-shaped action |
| `GenericAPIView` + mixins | the mixins you actually want | CRUD over a queryset, with explicit URLs and only some verbs |
| `ViewSet` | actions, no queryset assumptions | a group of custom actions routed together |
| `ModelViewSet` | almost nothing | standard CRUD on one model — the default choice |

`GenericAPIView` adds `get_queryset()`, `get_object()`, `get_serializer()`, `get_serializer_class()`, pagination and filter backends. The mixins are thin wrappers that call those:

| Mixin | Action method | Verb + route | Calls |
| --- | --- | --- | --- |
| `ListModelMixin` | `list` | `GET /orders/` | `filter_queryset(get_queryset())` → paginate → serialize |
| `CreateModelMixin` | `create` | `POST /orders/` | `is_valid` → `perform_create` → `201` + `Location` |
| `RetrieveModelMixin` | `retrieve` | `GET /orders/{pk}/` | `get_object()` → serialize |
| `UpdateModelMixin` | `update` | `PUT /orders/{pk}/` | `get_object()` → `is_valid` → `perform_update` |
| `UpdateModelMixin` | `partial_update` | `PATCH /orders/{pk}/` | the same, with `partial=True` |
| `DestroyModelMixin` | `destroy` | `DELETE /orders/{pk}/` | `get_object()` → `perform_destroy` → `204` |

`perform_create` / `perform_update` / `perform_destroy` are the designed override points. Side effects belong there rather than in the serializer, which keeps the serializer a data-shape concern and the view the transaction and side-effect boundary. Anything slow — email, PDF rendering, a third-party call — belongs in a task queue instead of the request/response cycle: see [Celery](../Celery.md).

### ⚠️ `queryset` versus `get_queryset()`

```python
class OrderViewSet(viewsets.ModelViewSet):
    # ❌ evaluated ONCE, at import time
    queryset = Order.objects.filter(created_at__gte=timezone.now() - timedelta(days=7))

    # ❌ also broken: there is no request at class-body time
    queryset = Order.objects.filter(organization=request.user.organization)

    # ✅ built per request
    def get_queryset(self):
        return Order.objects.filter(
            organization=self.request.user.organization,
            created_at__gte=timezone.now() - timedelta(days=7),
        )
```

A class attribute is a **class-body expression**: `timezone.now()` is called when the module is imported, so a "last 7 days" endpoint actually serves the last 7 days *relative to the last deploy*, and the window drifts further every day the process stays up. The second form does not even import. Beyond the frozen value, a class-level queryset is a shared object whose result cache would be shared across requests if DRF did not defensively call `.all()` on it in `get_queryset()` — do not lean on that safety net; anything request-dependent goes in the method.

You still declare `queryset` (or pass `basename` to the router) because routers and schema generation introspect it for the model. `queryset = Order.objects.none()` plus a real `get_queryset()` keeps introspection working while guaranteeing the unscoped queryset can never be used by accident.

### 🎛️ One view, two serializers

```python
class OrderViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.action in ("list", "retrieve"):
            return OrderReadSerializer          # nested, annotated, wide
        return OrderWriteSerializer             # flat, strict, no computed fields

    @action(detail=True, methods=["post"], url_path="cancel",
            permission_classes=[IsAuthenticated, CanCancelOrder])
    def cancel(self, request, pk=None):
        order = self.get_object()               # scoped queryset + object permissions
        order.cancel()
        return Response(OrderReadSerializer(order).data)
```

Separate read and write serializers beat one serializer full of `if self.context["request"]…`. The read shape can be wide and nested without ever *accepting* those fields as input, which removes a whole class of property-level authorization bugs by construction.

### 🗺️ Routers, `basename` and lookups

| Knob | Effect |
| --- | --- |
| `router.register(r"orders", OrderViewSet)` | derives `basename` from `queryset.model`; **raises** if you only define `get_queryset()` — pass `basename="order"` |
| `basename` | the prefix for reverse names: `order-list`, `order-detail`, `order-cancel` |
| `@action(detail=True)` | `/orders/{pk}/cancel/`; `detail=False` gives `/orders/cancel/` |
| `lookup_field` / `lookup_url_kwarg` | swap `pk` for `slug` or `uuid`; the URL kwarg may differ from the model field |
| `DefaultRouter` vs `SimpleRouter` | the former adds an API-root view and `.json` format suffixes |
| `get_extra_actions()` | how the router discovers `@action` methods on the class |

> [!TIP]
> `@action` takes its own `permission_classes`, `throttle_classes` and `serializer_class`. A `POST /orders/{pk}/cancel/` that silently inherits the viewset's read permissions is a textbook broken-function-level-authorization finding.

---
## 🔑 Permissions — and the check that silently does not run

Permission classes answer two different questions at two different times:

| Hook | Signature | Runs |
| --- | --- | --- |
| `has_permission` | `(self, request, view)` | in `initial()`, for **every** request, before the handler |
| `has_object_permission` | `(self, request, view, obj)` | only from `check_object_permissions()` — which only `get_object()` calls |

```python
class IsOrgMember(BasePermission):
    def has_permission(self, request, view):
        return bool(request.user and request.user.is_authenticated)

    def has_object_permission(self, request, view, obj):
        return obj.organization_id == request.user.organization_id
```

```python
# ❌ has_object_permission NEVER runs here — nothing called get_object()
def retrieve(self, request, pk=None):
    order = Order.objects.get(pk=pk)               # any pk, any tenant
    return Response(OrderSerializer(order).data)

# ✅ either go through get_object()...
def retrieve(self, request, pk=None):
    order = self.get_object()                      # queryset filter + object perms
    return Response(OrderSerializer(order).data)

# ✅ ...or check explicitly when you must fetch differently
order = get_object_or_404(Order, reference=ref)
self.check_object_permissions(request, order)
```

This is the most important single fact about DRF permissions, and it has a second half: **`list` never calls `get_object()`**, so no object-level permission can ever protect a collection endpoint. Object permissions filter one object; they cannot filter a set. What keeps another tenant's rows out of a list response is the queryset:

```python
def get_queryset(self):
    # the one line that prevents BOLA on the collection
    return Order.objects.filter(organization=self.request.user.organization)
```

Scope the queryset and object-level safety follows for free, because `get_object()` looks the object up *inside* `filter_queryset(get_queryset())` — a foreign id then 404s before any permission class is consulted. That is also why 404 is the right answer for "exists, but not yours": a `403` confirms the id exists. The vulnerability class and the test pattern that catches it are catalogued in [API Security](../../Security/API%20Security.md); the model-level question of where policy lives (RBAC, ABAC, per-object backends) is in [Access Control](../../Security/AccessControl.md) and [Authorization](../Authorization.md).

### 🚪 Deny by default, and composition

```python
# settings.py — applies to every view that does not override it
REST_FRAMEWORK = {
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
}
```

DRF's out-of-the-box default is `AllowAny`. Leaving it there means every new view is public until somebody remembers to lock it, and the review that catches that omission is the one that never happens. Set `IsAuthenticated` globally and open individual views deliberately.

| Class | Grants | Foot-gun |
| --- | --- | --- |
| `AllowAny` | everyone | the default — and it never validates the token that *was* sent |
| `IsAuthenticated` | any logged-in user | says nothing about *which* objects; pair it with queryset scoping |
| `IsAdminUser` | `is_staff` | `is_staff` is one boolean, not a role model |
| `IsAuthenticatedOrReadOnly` | anonymous `GET`/`HEAD`/`OPTIONS`, authenticated writes | **your whole dataset becomes public**. Fine for a blog, a breach for a tenanted API |
| `DjangoModelPermissions` | Django `add`/`change`/`delete` perms | requires a `queryset`, ignores per-object scope, and maps nothing for `GET` by default |
| `DjangoObjectPermissions` | per-object model perms | needs a backend such as django-guardian |

Permissions compose with `&`, `|` and `~`, implemented by wrapper classes that forward both hooks:

```python
permission_classes = [IsAuthenticated & (IsOwner | IsOrgAdmin)]
```

> [!WARNING]
> A list of permission classes is an **AND** — every class must return `True`. `|` is where the subtlety lives: the OR wrapper evaluates `has_permission` on its operands and, separately, `has_object_permission` on them, so a class that passes the first hook and fails the second can make a composed expression fail in ways your sketched truth table did not predict. Test composed permissions with real requests instead of reasoning about them.

Permission classes also drive the `OPTIONS` metadata and the browsable form. Whether an unauthenticated request gets `401` or `403` is decided by the *authenticator*, not the permission: DRF returns `401` only if an auth class supplies a `WWW-Authenticate` header, which `TokenAuthentication` does and `SessionAuthentication` does not.

---
## 🪪 Authentication classes

`authentication_classes` is an ordered list. DRF tries each one until it returns a `(user, auth)` tuple; a class returning `None` means "not my credential type", not "rejected". Only raising `AuthenticationFailed` rejects.

| Class | Credential | Notes |
| --- | --- | --- |
| `SessionAuthentication` | Django session cookie | first-party browser clients; **enforces CSRF** on unsafe methods |
| `BasicAuthentication` | `Authorization: Basic` | tests and internal tools only — a password on every request |
| `TokenAuthentication` | `Authorization: Token <key>` | one DB row per user, never expires, revoked by deletion |
| simplejwt's `JWTAuthentication` | `Authorization: Bearer <jwt>` | stateless, cannot be revoked before `exp` — keep access tokens in minutes |
| `RemoteUserAuthentication` | an upstream header | safe only if the proxy always overwrites that header |

Ordering is a performance choice, not a security one: put the cheap common case first. The Django-side mechanics — user models, password hashing, session handling — are in [Authentication](../Authentication.md), delegated third-party access in [OAuth](../OAuth.md); and if you would rather not hand-write registration, login, activation and password-reset endpoints, [Djoser](./Djoser.md) ships them as viewsets to mount and then re-audit against your own permission defaults.

> [!IMPORTANT]
> Mixing `SessionAuthentication` with a token class on the same endpoint is how CSRF protection gets bypassed by accident. When a browser holds a session cookie, `SessionAuthentication` matches and CSRF applies — but a request crafted to match the token class instead never goes through that check. Keep cookie-authenticated and token-authenticated surfaces separate, and keep the settings half of this (`SECURE_*`, cookie flags, `ALLOWED_HOSTS`, `DEBUG`) in one audited place: [Security Settings](../SecuritySettings.md).

---
## 📄 Pagination, filtering and ordering

An unbounded list endpoint is the cheapest denial-of-service in any API: one client, one request, every row.

| Style | Class | Cost | Use when |
| --- | --- | --- | --- |
| Page number | `PageNumberPagination` | `LIMIT/OFFSET` — deep pages degrade linearly | small tables, UIs that show page numbers |
| Limit/offset | `LimitOffsetPagination` | the same, client-controlled | ad-hoc clients; **set `max_limit`** |
| Cursor | `CursorPagination` | keyset — constant cost at any depth | large or growing tables, feeds, exports |

```python
class OrderCursorPagination(CursorPagination):
    page_size = 50
    max_page_size = 200
    ordering = "-created_at"        # MUST be indexed, and should break ties uniquely
```

`CursorPagination` buys constant cost at the price of "jump to page 7", and it needs a stable ordering: ties in the ordering column make rows appear twice or disappear between pages, so order by something unique or append the primary key. `PageNumberPagination` with `page_size_query_param` set and `max_page_size` unset is unrestricted resource consumption with extra steps.

Filter backends run inside `filter_queryset()`, before pagination:

```python
class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ["status", "customer"]
    search_fields = ["reference", "customer__name"]
    ordering_fields = ["created_at", "total"]     # an allowlist, never "__all__"
    ordering = ["-created_at"]                    # deterministic default
```

- `ordering_fields = "__all__"` lets a client sort by any column, indexed or not. `ORDER BY total` over a million rows with no index is a full sort in the database on every request — the cheapest way for an anonymous client to burn your CPU. Keep the allowlist to indexed columns, and confirm the index is actually used before adding one: [Query Execution Plans](../../Database/Query%20Execution%20Plans.md) covers reading the plan.
- A queryset with no deterministic `ORDER BY` returns rows in whatever order the database finds convenient, so paginated results can repeat or skip rows between pages. Always set a default `ordering`.
- `SearchFilter` builds `icontains` chains — a sequential scan with a leading wildcard. Past a few thousand rows, move to `SearchVector` or trigram matching with a GIN index.

---
## 🤝 Content negotiation, renderers and errors

`perform_content_negotiation` matches `Accept` against `renderer_classes` in order, and the **first renderer wins** when the client sends `Accept: */*`.

```python
REST_FRAMEWORK = {
    "DEFAULT_RENDERER_CLASSES": ["rest_framework.renderers.JSONRenderer"],
    "DEFAULT_PARSER_CLASSES": ["rest_framework.parsers.JSONParser"],
    "EXCEPTION_HANDLER": "myapp.api.exceptions.handler",
}
```

> [!WARNING]
> `BrowsableAPIRenderer` is in DRF's defaults. In production it does two harmful things: it renders an HTML form for every endpoint, evaluating your related fields' querysets to populate `<select>` dropdowns — an accidental full-table dump on any large relation — and it publishes your whole API surface to anyone with a browser. Ship JSON only; if you want interactive docs, serve a schema-driven UI behind authentication.

DRF converts anything subclassing `APIException` into a `Response`; everything else propagates as a real `500`. Django's `Http404` and `PermissionDenied` are translated too. A custom `EXCEPTION_HANDLER` is how you get one error envelope across the whole API:

```python
def handler(exc, context):
    response = exception_handler(exc, context)          # DRF's default first
    if response is None:
        return None                                     # let genuine 500s be 500s
    request_id = getattr(context["request"], "request_id", None)
    response.data = {
        "error": {
            "code": getattr(exc, "default_code", "error"),   # stable, machine-readable
            "message": "Request failed",                     # never str(exc)
            "details": response.data,                        # field errors, as-is
            "request_id": request_id,                        # correlate with the logs
        }
    }
    return response
```

Return a `code` clients can branch on and keep the human message generic — an ORM or database message in an error body hands over table and column names for free. Schema generation (`rest_framework.schemas`, or drf-spectacular for anything real) reads your serializers, so explicit fields and typed `SerializerMethodField`s are what make the generated OpenAPI document usable; and an accurate schema is what lets you fuzz the API against it.

---
## 🧪 Testing the pipeline

```python
from rest_framework.test import APIClient, APITestCase

class OrderAPITests(APITestCase):
    def setUp(self):
        self.client = APIClient()
        self.alice, self.bob = make_user("alice"), make_user("bob")   # different orgs
        self.order = Order.objects.create(organization=self.alice.organization)

    def test_cross_tenant_read_is_404(self):
        self.client.force_authenticate(self.bob)      # bypasses the auth classes
        r = self.client.get(f"/api/orders/{self.order.pk}/")
        self.assertEqual(r.status_code, 404)          # not 403 — do not confirm the id

    def test_list_query_count_is_constant(self):
        self.client.force_authenticate(self.alice)
        make_orders(self.alice.organization, n=50, tags_each=3)
        with self.assertNumQueries(4):                # count + page + 2 prefetches
            self.client.get("/api/orders/")
```

- `APIClient` speaks the API: `client.post(url, data, format="json")` goes through the parsers, unlike Django's test client, which form-encodes by default.
- `force_authenticate(user)` sets `request.user` and **skips the authentication classes entirely**. That is exactly right for permission tests and exactly wrong for testing the auth classes themselves — those need a real credential. The standalone `force_authenticate(request, user=...)` helper does the same thing for `APIRequestFactory` requests.
- `assertNumQueries` is the regression test for the N+1 section above. Pin it on every list endpoint.
- Tests that pass alone and fail in a suite are usually ordering, shared state or time; the taxonomy and the fixes are in [Flaky Tests](../../Python/FlakyTest.md).

---
## 🚨 Common mistakes

| Mistake | Why it hurts | Fix |
| --- | --- | --- |
| `fields = "__all__"` | every future column becomes readable *and* writable, including `is_staff` | list fields explicitly; `read_only_fields` for derived ones |
| Relying on `has_object_permission` for `list` | it never runs on a collection, so other tenants' rows are returned | scope `get_queryset()`; treat object perms as the second layer |
| A custom lookup that skips `get_object()` | object permissions are skipped silently, with no error anywhere | call `self.check_object_permissions(request, obj)` |
| `queryset = Model.objects.filter(created_at__gte=now() - …)` | evaluated at import, so the window freezes at deploy time | move it into `get_queryset()` |
| Owner taken from `request.data` | any client sets any owner | `serializer.save(user=request.user)` |
| Leaving `DEFAULT_PERMISSION_CLASSES` at `AllowAny` | every new endpoint is public until someone notices | deny by default, open per view |
| `IsAuthenticatedOrReadOnly` on tenanted data | the entire dataset is readable by anyone with an account | per-tenant queryset scoping |
| Relation traversal in a serializer field | N+1: one query per row per field | `select_related` / `prefetch_related` / `annotate` in `get_queryset()` |
| `obj.items.count()` in a `SerializerMethodField` | a new queryset defeats the prefetch and the query returns | `annotate(Count(…))` or `Prefetch(to_attr=…)` |
| `many=True` plus `save()` on existing instances | `NotImplementedError` — `ListSerializer.update` does not exist | a `list_serializer_class` keyed on an id |
| Writable nested serializer with no `create()` | it raises at runtime, or the nested payload is dropped | explicit `create()`/`update()` inside a transaction |
| `validate()` indexing `attrs` on PATCH | `KeyError` for every field the client did not send | `attrs.get(k, getattr(self.instance, k, None))` |
| `ordering_fields = "__all__"` | a client-chosen sort on an unindexed column is free CPU exhaustion | allowlist indexed columns only |
| No `max_page_size`, no default `ordering` | one request pulls the table; pages repeat and skip rows | cap the page size and always order deterministically |
| `BrowsableAPIRenderer` left in production | HTML forms enumerate related tables and publish the API surface | `JSONRenderer` only |
| `str(exc)` in the error body | schema, SQL and file paths handed to the caller | stable code, generic message, correlation id |
| `request.POST` on a JSON endpoint | an empty dict — and reading `request.body` first breaks `request.data` | always `request.data` |

---
## 🧠 Summary

| Area | Takeaway |
| --- | --- |
| Pipeline | `as_view` → `dispatch` → `initialize_request` → `initial` (negotiate, version, authenticate, permissions, throttles) → handler → `finalize_response` |
| Request | `request.data` is parser-driven and lazy, `request.POST` is empty for JSON, and the body can be read once |
| Authentication | Lazy on first `request.user` access — an `AllowAny` view never validates the token it was sent |
| Serializers | `is_valid` → `to_internal_value` → field validators → `validate_<field>` → `validate` → `Meta.validators`, in that order |
| Input safety | Explicit `fields`, `read_only_fields`, owner from the token; unknown keys are dropped without a word |
| PATCH | `partial=True` only makes fields optional: defaults stop applying and `validate()` must consult `self.instance` |
| Nesting | `many=True` is a `ListSerializer`; writable nesting needs your own `create()`, and `update` does not exist |
| Performance | Serializer fields walk relations one row at a time — fix it in `get_queryset()` and pin it with `assertNumQueries` |
| Views | `get_queryset()` per request, never a class attribute holding a computed value; side effects in `perform_*` |
| Permissions | `has_permission` always, `has_object_permission` only via `get_object()`; queryset scoping is the real control |
| Limits | Capped page size, cursor pagination on big tables, an ordering allowlist of indexed columns |
| Output | JSON renderer only, one error envelope with a stable code and a correlation id |

---
## 📚 References

- [DRF — requests](https://www.django-rest-framework.org/api-guide/requests/) · [serializers](https://www.django-rest-framework.org/api-guide/serializers/) · [validators](https://www.django-rest-framework.org/api-guide/validators/)
- [DRF — generic views](https://www.django-rest-framework.org/api-guide/generic-views/) · [viewsets](https://www.django-rest-framework.org/api-guide/viewsets/) · [routers](https://www.django-rest-framework.org/api-guide/routers/)
- [DRF — permissions](https://www.django-rest-framework.org/api-guide/permissions/) · [authentication](https://www.django-rest-framework.org/api-guide/authentication/) · [pagination](https://www.django-rest-framework.org/api-guide/pagination/) · [exceptions](https://www.django-rest-framework.org/api-guide/exceptions/) · [testing](https://www.django-rest-framework.org/api-guide/testing/)
- [Django — class-based views reference](https://docs.djangoproject.com/en/stable/ref/class-based-views/) · [testing tools](https://docs.djangoproject.com/en/stable/topics/testing/tools/)
- [django-filter with DRF](https://django-filter.readthedocs.io/en/stable/guide/rest_framework.html) · [drf-spectacular](https://drf-spectacular.readthedocs.io/en/latest/)
- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
