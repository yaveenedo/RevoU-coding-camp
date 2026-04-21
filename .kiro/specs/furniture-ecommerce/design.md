# Design Document: Furniture E-Commerce

## Overview

A server-side rendered furniture e-commerce web application built with Django. Visitors can browse the product catalog without authentication. Purchasing, cart management, and reviews are restricted to users authenticated via Google OAuth 2.0. Payments are processed through the Midtrans Snap payment gateway.

The application follows a classic Django monolith pattern: URL routing → views → templates, with the Django ORM handling all database access. No REST API layer is introduced; all interactions are form submissions and server-rendered page responses, with targeted JavaScript for dynamic UI (cart quantity updates, Midtrans Snap popup, form validation feedback).

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser                              │
│  Django Templates (HTML) + Tailwind CSS + JavaScript        │
└────────────────────┬────────────────────────────────────────┘
                     │ HTTP
┌────────────────────▼────────────────────────────────────────┐
│                    Django Application                        │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │ catalog  │  │  cart    │  │  orders  │  │  reviews  │  │
│  │   app    │  │   app    │  │   app    │  │    app    │  │
│  └──────────┘  └──────────┘  └──────────┘  └───────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              accounts app (django-allauth)           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Django ORM                         │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼──────┐         ┌────────▼───────┐
│  SQLite /    │         │  External APIs  │
│  PostgreSQL  │         │                 │
└──────────────┘         │ Google OAuth2   │
                         │ Midtrans Snap   │
                         └─────────────────┘
```

### Key Architectural Decisions

- **Monolith over microservices**: Django's built-in ORM, URL routing, and template engine are sufficient for this scope. No separate API server is needed.
- **django-allauth for OAuth**: Handles the full Google OAuth 2.0 flow (redirect, callback, user creation/retrieval, session management) with minimal custom code.
- **Midtrans Snap (popup mode)**: The backend creates a transaction token via the `midtransclient` Python library; the frontend loads the Snap.js popup. This avoids a full-page redirect and keeps the user on the site.
- **Server-side rendering**: All pages are rendered by Django templates. JavaScript is used only for dynamic interactions (quantity updates, Snap popup trigger, inline form validation).
- **Session-based cart**: Cart items are stored in the database (not the session), keyed to the authenticated user, ensuring persistence across sessions and devices.

---

## Components and Interfaces

### Django App Structure

```
furniture_ecommerce/          # Django project root
├── furniture_ecommerce/      # Project package
│   ├── settings.py
│   ├── urls.py               # Root URL configuration
│   └── wsgi.py
├── catalog/                  # Product browsing
│   ├── models.py             # Product
│   ├── views.py              # ProductListView, ProductDetailView
│   ├── urls.py
│   └── templates/catalog/
│       ├── product_list.html
│       └── product_detail.html
├── cart/                     # Cart management
│   ├── models.py             # Cart, CartItem
│   ├── views.py              # CartDetailView, AddToCartView, UpdateCartView, RemoveFromCartView
│   ├── urls.py
│   └── templates/cart/
│       └── cart_detail.html
├── orders/                   # Order creation and tracking
│   ├── models.py             # Order, OrderItem
│   ├── views.py              # CheckoutView, MidtransCallbackView
│   ├── urls.py
│   └── templates/orders/
│       └── checkout.html
├── reviews/                  # Product reviews
│   ├── models.py             # Review
│   ├── views.py              # SubmitReviewView
│   ├── urls.py
│   └── templates/reviews/
│       └── (included in product_detail.html)
├── accounts/                 # Auth configuration (thin wrapper over allauth)
│   ├── urls.py               # Delegates to allauth URLs
│   └── templates/account/
│       ├── login.html        # Custom allauth login template
│       └── social_signup.html
├── templates/
│   └── base.html             # Shared base template with Tailwind, nav
└── static/
    ├── css/
    │   └── output.css        # Compiled Tailwind CSS
    └── js/
        ├── cart.js           # Cart quantity update interactions
        ├── checkout.js       # Midtrans Snap popup trigger
        └── validation.js     # Client-side form validation feedback
```

### URL Routing

| URL Pattern | View | Name | Auth Required |
|---|---|---|---|
| `/` | `ProductListView` | `catalog:product_list` | No |
| `/products/<slug>/` | `ProductDetailView` | `catalog:product_detail` | No |
| `/cart/` | `CartDetailView` | `cart:detail` | Yes |
| `/cart/add/<product_id>/` | `AddToCartView` | `cart:add` | Yes (redirect to login if not) |
| `/cart/update/<item_id>/` | `UpdateCartView` | `cart:update` | Yes |
| `/cart/remove/<item_id>/` | `RemoveFromCartView` | `cart:remove` | Yes |
| `/checkout/` | `CheckoutView` | `orders:checkout` | Yes |
| `/checkout/callback/` | `MidtransCallbackView` | `orders:callback` | No (Midtrans server-to-server) |
| `/reviews/submit/<product_id>/` | `SubmitReviewView` | `reviews:submit` | Yes |
| `/accounts/login/` | allauth login | `account_login` | No |
| `/accounts/logout/` | allauth logout | `account_logout` | Yes |
| `/accounts/google/login/` | allauth Google redirect | `google_login` | No |
| `/accounts/google/login/callback/` | allauth Google callback | `google_callback` | No |

### View Interfaces

**`AddToCartView`** (POST only)
- If unauthenticated: redirect to `account_login` with `next` parameter
- If authenticated: get or create `Cart` for user, upsert `CartItem`, redirect to `cart:detail`

**`CheckoutView`** (GET + POST)
- GET: render checkout page with cart summary; call `midtransclient.Snap.create_transaction_token()` and embed token in template context
- POST: not used (payment result comes via Midtrans callback)

**`MidtransCallbackView`** (POST only, no auth)
- Verify Midtrans notification signature
- On success: create `Order` + `OrderItem` records, clear cart, return HTTP 200
- On failure/cancel: return HTTP 200 (Midtrans requires 200 for all callbacks)

**`SubmitReviewView`** (POST only)
- Verify user has a completed `Order` containing the product
- Validate review text is non-empty
- Check no existing review for this user/product/order combination
- Save `Review`, redirect to `catalog:product_detail`

---

## Data Models

```python
# catalog/models.py
class Product(models.Model):
    name        = models.CharField(max_length=255)
    slug        = models.SlugField(unique=True)
    description = models.TextField()
    price       = models.DecimalField(max_digits=12, decimal_places=2)
    image       = models.ImageField(upload_to='products/')
    is_active   = models.BooleanField(default=True)
    created_at  = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']


# cart/models.py
class Cart(models.Model):
    user       = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

class CartItem(models.Model):
    cart       = models.ForeignKey(Cart, on_delete=models.CASCADE, related_name='items')
    product    = models.ForeignKey(Product, on_delete=models.CASCADE)
    quantity   = models.PositiveIntegerField(default=1)

    class Meta:
        unique_together = ('cart', 'product')

    @property
    def subtotal(self):
        return self.product.price * self.quantity


# orders/models.py
class Order(models.Model):
    class Status(models.TextChoices):
        PENDING  = 'pending',  'Pending'
        PAID     = 'paid',     'Paid'
        FAILED   = 'failed',   'Failed'

    user              = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT)
    midtrans_order_id = models.CharField(max_length=100, unique=True)
    total_price       = models.DecimalField(max_digits=12, decimal_places=2)
    status            = models.CharField(max_length=20, choices=Status.choices, default=Status.PENDING)
    created_at        = models.DateTimeField(auto_now_add=True)
    updated_at        = models.DateTimeField(auto_now=True)

class OrderItem(models.Model):
    order      = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='items')
    product    = models.ForeignKey(Product, on_delete=models.PROTECT)
    quantity   = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=12, decimal_places=2)  # snapshot at purchase time


# reviews/models.py
class Review(models.Model):
    user       = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    product    = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='reviews')
    order      = models.ForeignKey(Order, on_delete=models.CASCADE)
    text       = models.TextField()
    is_approved = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('user', 'product', 'order')
```

### Entity Relationship Diagram

```
User (django-allauth)
 │
 ├──(1:1)── Cart ──(1:N)── CartItem ──(N:1)── Product
 │
 ├──(1:N)── Order ──(1:N)── OrderItem ──(N:1)── Product
 │
 └──(1:N)── Review ──(N:1)── Product
                  └──(N:1)── Order
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Catalog displays all active products regardless of auth state

*For any* set of active products and any request (authenticated or unauthenticated), the catalog view response should contain all active products.

**Validates: Requirements 1.1**

---

### Property 2: Product card renders all required fields

*For any* product, the rendered product card HTML should contain the product's name, image URL, description, and price.

**Validates: Requirements 1.2**

---

### Property 3: Search/filter returns exactly matching products

*For any* search query and product catalog, every product in the result set should match the query criteria, and no product matching the criteria should be absent from the result set.

**Validates: Requirements 1.3**

---

### Property 4: OAuth callback creates or retrieves user and establishes session

*For any* valid Google OAuth callback payload (new or existing Google identity), the resulting Django session should be authenticated and the user record should match the Google identity's email and name.

**Validates: Requirements 2.3**

---

### Property 5: Unauthenticated add-to-cart redirects to login

*For any* product, a POST request to the add-to-cart endpoint while unauthenticated should return a redirect response pointing to the login page.

**Validates: Requirements 3.2**

---

### Property 6: Authenticated add-to-cart adds product to cart

*For any* product and authenticated user, after a successful add-to-cart POST, the product should appear in the user's cart with at least quantity 1.

**Validates: Requirements 3.4**

---

### Property 7: Cart quantity upsert invariant

*For any* product and user cart state, adding a product that already exists in the cart should increment its quantity by exactly 1; adding a product not yet in the cart should create a new cart item with quantity 1.

**Validates: Requirements 4.2**

---

### Property 8: Cart item removal removes the item

*For any* cart item belonging to a user, after a remove request, that item should no longer appear in the user's cart.

**Validates: Requirements 4.4**

---

### Property 9: Cart total equals arithmetic sum

*For any* cart with any set of items (each with a price and quantity), the displayed cart total should equal the sum of (unit_price × quantity) for all items.

**Validates: Requirements 4.5**

---

### Property 10: Midtrans token request contains correct cart total and user details

*For any* authenticated user and any non-empty cart, the Midtrans Snap token creation request payload should contain a `gross_amount` equal to the cart total and `customer_details` matching the user's name and email.

**Validates: Requirements 5.1**

---

### Property 11: Successful Midtrans callback creates paid order and clears cart

*For any* valid Midtrans success notification payload, an Order record should be created with `status=paid` associated with the correct user, and the user's cart should be empty after processing.

**Validates: Requirements 5.3, 5.5**

---

### Property 12: Product detail displays all approved reviews

*For any* product with any set of reviews, the product detail page should display all reviews where `is_approved=True`, each including the reviewer's name and review text.

**Validates: Requirements 6.1**

---

### Property 13: Review submission succeeds for purchasers

*For any* user who has a completed (paid) order containing a product, submitting a non-empty review for that product should create a Review record associated with that user and product.

**Validates: Requirements 6.2**

---

### Property 14: Review submission rejected for non-purchasers

*For any* user who has no completed order containing a product, submitting a review for that product should be rejected with an appropriate error message and no Review record should be created.

**Validates: Requirements 6.3**

---

### Property 15: One review per user-product-order combination

*For any* user-product-order combination that already has a review, attempting to submit a second review should be rejected and the review count for that combination should remain 1.

**Validates: Requirements 6.5**

---

## Error Handling

### Authentication Errors

- **OAuth denial / error**: django-allauth catches the OAuth error response and redirects to `/accounts/login/` with an error message in the Django messages framework. The custom `login.html` template renders this message.
- **Session expiry**: Django's `@login_required` decorator (or `LoginRequiredMixin`) redirects expired sessions to the login page with `?next=<original_url>`.

### Cart Errors

- **Add to cart while unauthenticated**: `AddToCartView` checks `request.user.is_authenticated`; if false, redirects to `settings.LOGIN_URL` with `next` parameter.
- **Product not found**: `get_object_or_404(Product, pk=product_id, is_active=True)` returns HTTP 404.
- **Quantity set to zero**: `UpdateCartView` deletes the `CartItem` when `quantity <= 0`.

### Payment Errors

- **Midtrans token creation failure**: Wrap `snap.create_transaction_token()` in a try/except; on exception, redirect to cart with an error message via Django messages.
- **Midtrans callback signature mismatch**: `MidtransCallbackView` verifies the notification using `midtransclient`'s built-in signature verification. On mismatch, return HTTP 400 and log the event.
- **Failed/cancelled payment**: Callback handler checks `transaction_status`; on `failure` or `cancel`, the cart is retained and a failure message is stored in the session for display on the next page load.
- **Duplicate callback**: `Order` has `unique=True` on `midtrans_order_id`; duplicate callbacks are handled with `get_or_create` to avoid duplicate orders.

### Review Errors

- **Empty review text**: `SubmitReviewView` validates `text.strip()` is non-empty; returns form with validation error if not.
- **User has not purchased product**: View queries `Order.objects.filter(user=request.user, status='paid', items__product=product)`; if empty queryset, returns error message.
- **Duplicate review**: Database `unique_together` constraint on `(user, product, order)` prevents duplicates; view checks for existing review before saving and returns an error message.

---

## Testing Strategy

### Dual Testing Approach

Both unit/example-based tests and property-based tests are used. Unit tests cover specific examples, integration points, and edge cases. Property-based tests verify universal invariants across generated inputs.

### Property-Based Testing

The feature involves pure business logic functions (cart total calculation, review eligibility checks, search/filter logic, Midtrans payload construction) that are well-suited for property-based testing.

**Library**: [`hypothesis`](https://hypothesis.readthedocs.io/) for Python.

**Configuration**: Each property test runs a minimum of 100 iterations (`@settings(max_examples=100)`).

**Tag format**: Each property test is tagged with a comment:
`# Feature: furniture-ecommerce, Property <N>: <property_text>`

**Properties to implement as property-based tests**:

| Property | Test Focus | Hypothesis Strategy |
|---|---|---|
| P1: Catalog displays all products | `ProductListView` response | `st.lists(product_strategy)` |
| P2: Product card renders all fields | Template rendering | `product_strategy` |
| P3: Search returns matching products | `ProductListView` with `?q=` | `st.text()` + `st.lists(product_strategy)` |
| P5: Unauthenticated add redirects | `AddToCartView` | `product_strategy` |
| P6: Authenticated add adds to cart | `AddToCartView` | `product_strategy` + `user_strategy` |
| P7: Cart quantity upsert invariant | `Cart.add_product()` | `product_strategy` + `st.integers(min_value=1)` |
| P8: Cart item removal | `RemoveFromCartView` | `cart_item_strategy` |
| P9: Cart total calculation | `Cart.total` property | `st.lists(cart_item_strategy, min_size=1)` |
| P10: Midtrans payload correctness | `build_midtrans_payload()` | `cart_strategy` + `user_strategy` |
| P11: Callback creates order + clears cart | `MidtransCallbackView` | `midtrans_notification_strategy` |
| P12: Product detail shows approved reviews | `ProductDetailView` | `st.lists(review_strategy)` |
| P13: Review submission for purchasers | `SubmitReviewView` | `user_strategy` + `product_strategy` |
| P14: Review rejection for non-purchasers | `SubmitReviewView` | `user_strategy` + `product_strategy` |
| P15: One review per combination | `SubmitReviewView` | `user_strategy` + `product_strategy` |

### Unit / Example-Based Tests

- **OAuth redirect**: Verify GET `/accounts/google/login/` returns 302 to `accounts.google.com`.
- **OAuth error handling**: Mock allauth callback with error; verify redirect to login with error message.
- **Logout**: Verify POST `/accounts/logout/` invalidates session and redirects to homepage.
- **Empty cart display**: Verify empty cart page renders empty-cart message and browse prompt.
- **Zero quantity removes item**: Verify updating cart item quantity to 0 deletes the item.
- **Midtrans Snap UI rendered**: Verify checkout page includes Snap.js script tag and transaction token.
- **Failed payment retains cart**: Mock failed Midtrans callback; verify cart items unchanged.
- **Empty review rejected**: Verify submitting whitespace-only review text returns validation error.

### Integration Tests

- **Google OAuth full flow**: End-to-end test using a mocked Google OAuth server (or `django-allauth` test utilities).
- **Midtrans callback signature verification**: Test with valid and invalid HMAC signatures.
- **Database constraints**: Verify `unique_together` on `Review` and `CartItem` are enforced at the DB level.

### Test Organization

```
tests/
├── catalog/
│   ├── test_views.py          # Unit + property tests for catalog views
│   └── test_models.py
├── cart/
│   ├── test_views.py          # Unit + property tests for cart views
│   ├── test_models.py         # Property tests for cart total, upsert
│   └── strategies.py          # Hypothesis strategies for cart data
├── orders/
│   ├── test_views.py          # Unit + property tests for checkout/callback
│   └── test_models.py
├── reviews/
│   ├── test_views.py          # Unit + property tests for review submission
│   └── test_models.py
└── conftest.py                # Shared fixtures, Hypothesis profiles
```
