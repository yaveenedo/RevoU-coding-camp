# Implementation Plan: Furniture E-Commerce

## Overview

Implement a server-side rendered Django monolith for a furniture e-commerce store. The build proceeds in layers: project scaffolding → data models → authentication → catalog → cart → checkout/payments → reviews → JavaScript → tests. Each layer is independently testable before the next begins.

## Tasks

- [x] 1. Django project scaffolding and settings
  - Create the Django project with `django-admin startproject furniture_ecommerce`
  - Create the five apps: `catalog`, `cart`, `orders`, `reviews`, `accounts`
  - Configure `settings.py`: `INSTALLED_APPS`, `DATABASES` (SQLite for dev), `STATIC_ROOT`, `MEDIA_ROOT`, `LOGIN_URL = '/accounts/login/'`, `LOGIN_REDIRECT_URL = '/'`
  - Add `django-allauth`, `midtransclient`, `Pillow`, `hypothesis` to `requirements.txt`
  - Configure `AUTHENTICATION_BACKENDS`, `SITE_ID`, and allauth settings in `settings.py` (`SOCIALACCOUNT_PROVIDERS` for Google, `ACCOUNT_EMAIL_REQUIRED = True`, `SOCIALACCOUNT_AUTO_SIGNUP = True`)
  - Set up Tailwind CSS: add `tailwind` and `django_browser_reload` to `INSTALLED_APPS`, create `theme` app, configure `TAILWIND_APP_NAME`, add `NPM_BIN_PATH`
  - Create `templates/base.html` with Tailwind CDN/output.css link, nav bar (logo, login/logout, cart icon), and `{% block content %}` slot
  - Wire root `urls.py` to include all app URL modules and `allauth.urls`
  - _Requirements: 7.1, 7.2, 7.3_

- [x] 2. Database models
  - [x] 2.1 Implement `catalog/models.py` — `Product` model
    - Fields: `name`, `slug` (unique), `description`, `price` (DecimalField), `image` (ImageField), `is_active`, `created_at`
    - `Meta.ordering = ['-created_at']`
    - _Requirements: 1.2_

  - [x] 2.2 Implement `cart/models.py` — `Cart` and `CartItem` models
    - `Cart`: OneToOneField to `AUTH_USER_MODEL`, timestamps
    - `CartItem`: ForeignKey to `Cart` and `Product`, `quantity` PositiveIntegerField, `unique_together = ('cart', 'product')`
    - `CartItem.subtotal` property: `self.product.price * self.quantity`
    - `Cart.total` property: `sum(item.subtotal for item in self.items.all())`
    - _Requirements: 4.1, 4.5_

  - [ ]* 2.3 Write property test for `Cart.total` calculation
    - **Property 9: Cart total equals arithmetic sum**
    - **Validates: Requirements 4.5**
    - Use `st.lists(cart_item_strategy, min_size=1)` to generate arbitrary cart states
    - Assert `cart.total == sum(item.product.price * item.quantity for item in items)`

  - [x] 2.4 Implement `orders/models.py` — `Order` and `OrderItem` models
    - `Order`: ForeignKey to user, `midtrans_order_id` (unique), `total_price`, `status` (TextChoices: pending/paid/failed), timestamps
    - `OrderItem`: ForeignKey to `Order` and `Product`, `quantity`, `unit_price` (price snapshot)
    - _Requirements: 5.3_

  - [x] 2.5 Implement `reviews/models.py` — `Review` model
    - Fields: ForeignKeys to user, `Product`, `Order`; `text`, `is_approved` (default True), `created_at`
    - `unique_together = ('user', 'product', 'order')`
    - _Requirements: 6.1, 6.5_

  - [x] 2.6 Generate and run all migrations
    - `python manage.py makemigrations` for all apps
    - `python manage.py migrate`
    - _Requirements: 7.1_

- [x] 3. Google OAuth authentication
  - [x] 3.1 Configure `accounts/urls.py` to delegate to `allauth.urls`
    - Include `django.contrib.auth.urls` and `allauth.urls` under `/accounts/`
    - _Requirements: 2.1, 2.2_

  - [x] 3.2 Create custom allauth templates
    - `accounts/templates/account/login.html`: extend `base.html`, render Django messages (for OAuth errors), show "Sign in with Google" button linking to `{% url 'google_login' %}`
    - `accounts/templates/account/social_signup.html`: extend `base.html`, render signup confirmation form
    - _Requirements: 2.2, 2.4_

  - [ ]* 3.3 Write unit tests for OAuth flows
    - Test GET `/accounts/google/login/` returns 302 toward Google
    - Test allauth callback with mocked error redirects to login with error message (Requirement 2.4)
    - Test POST `/accounts/logout/` invalidates session and redirects to `/` (Requirement 2.5)

- [x] 4. Product catalog views and templates
  - [x] 4.1 Implement `catalog/views.py`
    - `ProductListView` (ListView): queryset `Product.objects.filter(is_active=True)`, supports `?q=` GET param to filter by name/description (`icontains`)
    - `ProductDetailView` (DetailView): lookup by `slug`, prefetch `reviews` where `is_approved=True`
    - _Requirements: 1.1, 1.2, 1.3_

  - [x] 4.2 Create `catalog/urls.py` and wire into root `urls.py`
    - `''` → `ProductListView` as `catalog:product_list`
    - `'products/<slug:slug>/'` → `ProductDetailView` as `catalog:product_detail`
    - _Requirements: 1.1_

  - [x] 4.3 Create `catalog/templates/catalog/product_list.html`
    - Extend `base.html`; render search form (`?q=`); loop over `object_list` rendering product cards (name, image, description, price)
    - Shop button: if `user.is_authenticated` → POST form to `cart:add`; else → link to `account_login` with `?next=` (Requirement 3.1, 3.2)
    - _Requirements: 1.2, 3.1, 3.2, 3.3_

  - [x] 4.4 Create `catalog/templates/catalog/product_detail.html`
    - Extend `base.html`; render product fields; include approved reviews section; include review submission form (POST to `reviews:submit`) if user is authenticated
    - _Requirements: 1.2, 6.1_

  - [ ]* 4.5 Write property tests for catalog views
    - **Property 1: Catalog displays all active products regardless of auth state**
    - **Validates: Requirements 1.1**
    - Use `st.lists(product_strategy)` to create DB products; assert all appear in response

    - **Property 2: Product card renders all required fields**
    - **Validates: Requirements 1.2**
    - Use `product_strategy`; assert name, image URL, description, price in rendered HTML

    - **Property 3: Search/filter returns exactly matching products**
    - **Validates: Requirements 1.3**
    - Use `st.text()` + `st.lists(product_strategy)`; assert result set matches query criteria exactly

- [x] 5. Cart views and templates
  - [x] 5.1 Implement `cart/views.py`
    - `AddToCartView` (POST, `login_required`): `get_object_or_404(Product, pk=product_id, is_active=True)`; `get_or_create` Cart for user; upsert CartItem (increment quantity if exists, else create with qty=1); redirect to `cart:detail`
    - Unauthenticated path: check `request.user.is_authenticated`; if false, redirect to `settings.LOGIN_URL + '?next=' + request.path`
    - `CartDetailView` (GET, `LoginRequiredMixin`): fetch cart and items; render template
    - `UpdateCartView` (POST, `login_required`): update quantity; if `quantity <= 0` delete item; redirect to `cart:detail`
    - `RemoveFromCartView` (POST, `login_required`): delete CartItem; redirect to `cart:detail`
    - _Requirements: 3.2, 3.4, 4.1, 4.2, 4.3, 4.4_

  - [x] 5.2 Create `cart/urls.py` and wire into root `urls.py`
    - `'cart/'` → `CartDetailView` as `cart:detail`
    - `'cart/add/<int:product_id>/'` → `AddToCartView` as `cart:add`
    - `'cart/update/<int:item_id>/'` → `UpdateCartView` as `cart:update`
    - `'cart/remove/<int:item_id>/'` → `RemoveFromCartView` as `cart:remove`
    - _Requirements: 4.1_

  - [x] 5.3 Create `cart/templates/cart/cart_detail.html`
    - Extend `base.html`; if cart empty render empty-cart message + browse link (Requirement 4.6)
    - Otherwise render item table with quantity update forms and remove buttons; display cart total
    - Include "Proceed to Checkout" button linking to `orders:checkout`
    - _Requirements: 4.5, 4.6_

  - [ ]* 5.4 Write property tests for cart views and model
    - **Property 5: Unauthenticated add-to-cart redirects to login**
    - **Validates: Requirements 3.2**
    - Use `product_strategy`; assert POST by anonymous client returns redirect to login URL

    - **Property 6: Authenticated add-to-cart adds product to cart**
    - **Validates: Requirements 3.4**
    - Use `product_strategy` + `user_strategy`; assert product appears in cart after POST

    - **Property 7: Cart quantity upsert invariant**
    - **Validates: Requirements 4.2**
    - Use `product_strategy` + `st.integers(min_value=1)`; assert existing item increments by 1, new item starts at 1

    - **Property 8: Cart item removal removes the item**
    - **Validates: Requirements 4.4**
    - Use `cart_item_strategy`; assert item absent from cart after remove POST

- [x] 6. Checkpoint — Ensure all tests pass
  - Run `python manage.py test` (or `pytest`); ensure catalog and cart tests pass before proceeding.
  - Ensure all tests pass, ask the user if questions arise.

- [x] 7. Midtrans Snap payment integration
  - [x] 7.1 Implement `orders/views.py` — `CheckoutView`
    - `LoginRequiredMixin`; GET only
    - Fetch user's Cart; if empty redirect to `cart:detail` with message
    - Build Midtrans payload: `order_id` (UUID), `gross_amount` = cart total, `customer_details` from `request.user`
    - Call `snap.create_transaction_token(payload)` inside try/except; on exception redirect to `cart:detail` with error message
    - Pass `snap_token` and cart summary to template context
    - _Requirements: 5.1, 5.2_

  - [x] 7.2 Implement `orders/views.py` — `MidtransCallbackView`
    - POST only, no auth required (Midtrans server-to-server)
    - Parse JSON body; verify notification signature via `midtransclient`
    - On signature mismatch: return HTTP 400, log event
    - On `transaction_status == 'capture'` or `'settlement'`: use `get_or_create` on `Order` by `midtrans_order_id`; create `OrderItem` records from cart snapshot; set `status='paid'`; clear cart
    - On `failure`/`cancel`: set order status accordingly; retain cart; store failure message in session
    - Return HTTP 200 for all valid callbacks
    - _Requirements: 5.3, 5.4, 5.5_

  - [x] 7.3 Create `orders/urls.py` and wire into root `urls.py`
    - `'checkout/'` → `CheckoutView` as `orders:checkout`
    - `'checkout/callback/'` → `MidtransCallbackView` as `orders:callback`
    - _Requirements: 5.1_

  - [x] 7.4 Create `orders/templates/orders/checkout.html`
    - Extend `base.html`; render cart summary table with totals
    - Include Midtrans Snap.js script tag (`https://app.sandbox.midtrans.com/snap/snap.js`)
    - Embed `snap_token` in a `data-snap-token` attribute on the Pay button
    - _Requirements: 5.2_

  - [ ]* 7.5 Write property tests for checkout and callback
    - **Property 10: Midtrans token request contains correct cart total and user details**
    - **Validates: Requirements 5.1**
    - Use `cart_strategy` + `user_strategy`; mock `snap.create_transaction_token`; assert `gross_amount` equals cart total and `customer_details` match user

    - **Property 11: Successful Midtrans callback creates paid order and clears cart**
    - **Validates: Requirements 5.3, 5.5**
    - Use `midtrans_notification_strategy`; POST to callback view; assert `Order.status == 'paid'` and cart is empty

  - [ ]* 7.6 Write unit tests for payment edge cases
    - Test Midtrans token creation failure redirects to cart with error message
    - Test failed/cancelled callback retains cart items
    - Test duplicate callback handled via `get_or_create` (no duplicate Order)
    - Test callback with invalid signature returns HTTP 400

- [x] 8. Reviews
  - [x] 8.1 Implement `reviews/views.py` — `SubmitReviewView`
    - POST only, `login_required`
    - `get_object_or_404(Product, pk=product_id, is_active=True)`
    - Query `Order.objects.filter(user=request.user, status='paid', items__product=product)`; if empty, redirect to `catalog:product_detail` with error message (Requirement 6.3)
    - Validate `text.strip()` is non-empty; if empty, redirect with validation error (Requirement 6.4)
    - Check for existing `Review` for `(user, product, order)`; if exists, redirect with duplicate error (Requirement 6.5)
    - Save `Review`; redirect to `catalog:product_detail`
    - _Requirements: 6.2, 6.3, 6.4, 6.5_

  - [x] 8.2 Create `reviews/urls.py` and wire into root `urls.py`
    - `'reviews/submit/<int:product_id>/'` → `SubmitReviewView` as `reviews:submit`
    - _Requirements: 6.2_

  - [ ]* 8.3 Write property tests for review submission
    - **Property 12: Product detail displays all approved reviews**
    - **Validates: Requirements 6.1**
    - Use `st.lists(review_strategy)`; assert all `is_approved=True` reviews appear in `ProductDetailView` response

    - **Property 13: Review submission succeeds for purchasers**
    - **Validates: Requirements 6.2**
    - Use `user_strategy` + `product_strategy`; create paid order; assert Review record created after POST

    - **Property 14: Review submission rejected for non-purchasers**
    - **Validates: Requirements 6.3**
    - Use `user_strategy` + `product_strategy`; no paid order; assert no Review created and error message present

    - **Property 15: One review per user-product-order combination**
    - **Validates: Requirements 6.5**
    - Use `user_strategy` + `product_strategy`; submit review twice; assert review count remains 1

  - [ ]* 8.4 Write unit tests for review edge cases
    - Test empty/whitespace review text returns validation error (Requirement 6.4)
    - Test `unique_together` DB constraint enforced at DB level

- [x] 9. Checkpoint — Ensure all tests pass
  - Run full test suite; ensure orders and reviews tests pass before proceeding.
  - Ensure all tests pass, ask the user if questions arise.

- [x] 10. JavaScript — cart.js, checkout.js, validation.js
  - [x] 10.1 Implement `static/js/cart.js`
    - Attach `change` listeners to quantity `<input>` elements in `cart_detail.html`
    - On change: submit the parent update form via `fetch` (POST) and update the subtotal and total in the DOM without a full page reload
    - _Requirements: 7.4_

  - [x] 10.2 Implement `static/js/checkout.js`
    - Read `data-snap-token` from the Pay button
    - On button click: call `window.snap.pay(token, { onSuccess, onPending, onError, onClose })` callbacks
    - `onSuccess`: redirect to homepage with success message
    - `onError`/`onClose`: show inline error message without page reload
    - _Requirements: 5.2, 7.4_

  - [x] 10.3 Implement `static/js/validation.js`
    - Attach `submit` listener to the review form in `product_detail.html`
    - Validate that the review textarea is non-empty before submission; display inline error if empty
    - _Requirements: 6.4, 7.4_

- [x] 11. Static files and Tailwind CSS compilation setup
  - Configure `tailwind.config.js` with `content` paths covering all Django templates (`**/templates/**/*.html`)
  - Add `npm run build:css` script: `tailwindcss -i ./static/css/input.css -o ./static/css/output.css --minify`
  - Create `static/css/input.css` with `@tailwind base; @tailwind components; @tailwind utilities;`
  - Ensure `base.html` links to `{% static 'css/output.css' %}`
  - Configure `STATICFILES_DIRS` in `settings.py` to include the `static/` directory
  - _Requirements: 7.3_

- [x] 12. Hypothesis test infrastructure and shared strategies
  - Create `tests/conftest.py`: configure Hypothesis profile (`@settings(max_examples=100)`), shared Django test client fixtures
  - Create `tests/catalog/strategies.py`: `product_strategy` using `st.builds(Product, name=st.text(min_size=1), price=st.decimals(min_value=1, max_value=9999), ...)`
  - Create `tests/cart/strategies.py`: `cart_item_strategy`, `cart_strategy`
  - Create `tests/orders/strategies.py`: `midtrans_notification_strategy` for valid/invalid callback payloads
  - Create `tests/reviews/strategies.py`: `review_strategy`, `user_strategy`
  - _Requirements: 7.1_

- [x] 13. Final checkpoint — Ensure all tests pass
  - Run `pytest --tb=short` (or `python manage.py test`) across all test modules
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Checkpoints at tasks 6, 9, and 13 ensure incremental validation
- Property tests validate universal correctness invariants (Properties 1–15 from the design)
- Unit tests cover specific examples, edge cases, and integration points
- The Midtrans Snap popup flow means no full-page redirect on payment — `checkout.js` handles the popup lifecycle
- Run `npm run build:css` before collecting static files for production
