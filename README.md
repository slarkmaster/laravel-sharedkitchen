# MidwayCafe (RMS)

MidwayCafe, also called Midway Dine, is a restaurant management system built with Laravel 8. Guests browse the menu, reserve a table, and place delivery orders. Staff run the kitchen, orders, and site content from an admin panel.

The public site is branded **MidwayCafe**. Admin mail and invoices refer to the same system as **RMS**.

## Features

**Storefront**

- Home page with banners, an about section, and a YouTube embed
- Weekly specials split by meal: breakfast, lunch, and dinner
- Regular menu, chef profiles, and a table-reservation form
- Cart, percentage coupons, and extra charges such as shipping and VAT
- Checkout with cash on delivery or online payment
- Order history and invoice lookup
- Star ratings and a top-rated dishes page
- Confirmation emails, plus a PDF receipt with a QR code

**Admin**

- Dashboard of pending, processing, cancelled, and completed orders, plus cash versus online totals
- Menu, chef, banner, and about-page management
- Order workflow: approve with a delivery time, update location, cancel, or mark delivered
- Reservations, coupons, and charges
- Customers, super admins, sub admins, and delivery staff

## User roles

`users.usertype` decides where a signed-in person lands after `/redirects`.

| Value | Role | Lands on |
| --- | --- | --- |
| `0` | Customer | Storefront |
| `1` | Super admin | Admin dashboard |
| `2` | Delivery staff | Admin dashboard |
| `3` | Sub admin | Admin dashboard |

Admin pages check for usertype `1`. Delivery staff and sub admins are stored as users, but those routes are not limited to super admins in middleware.

## Order status

Orders live on cart rows, grouped by `invoice_no`. `carts.product_order` is the status:

| Value | Meaning |
| --- | --- |
| `no` | Still in the cart |
| `yes` | Placed, waiting for approval |
| `approve` | Approved and in process |
| `delivery` | Completed |
| `cancel` | Cancelled |

Online payments are recorded in `orders` (SSLCommerz / bKash, currency BDT). Cash on delivery skips that gateway and stores `Cash On Delivery` on the cart.

## Tech stack

- PHP `^7.3` or `^8.0`
- Laravel 8.54, Jetstream 2, Fortify, Sanctum, Livewire 2
- MySQL or MariaDB
- Tailwind CSS 2, Alpine.js, and Laravel Mix for the auth and profile screens
- Classic HTML templates in `public/assets` (storefront) and `public/admin` (admin)
- DomPDF and Simple QrCode for receipts
- SSLCommerz and bKash (sandbox) for online checkout
- SMTP, Mailgun, or Postmark for mail

## Project layout

```
app/Http/Controllers   Home, menu, cart, shipment, admin, SSLCommerz, bKash
app/Library/SslCommerz SSLCommerz client
app/Mail               Order, payment, reservation, and staff emails
resources/views        Storefront, admin, and mail templates
public/assets          Storefront CSS, JS, and images
public/admin           Admin template assets
database/migrations    Base schema (incomplete compared with the dump)
rms2.sql               Full MariaDB dump used by the running app
routes/web.php         All application routes
```

Most queries use the query builder (`DB::table`) rather than Eloquent. `User`, `Product`, `Cart`, and `order` models exist, but they do not cover chefs, reservations, coupons, charges, banners, or ratings.

## Requirements

- PHP 7.3+ or 8.x with extensions Laravel 8 expects (`openssl`, `pdo`, `mbstring`, `tokenizer`, `xml`, `ctype`, `json`, `bcmath`, `gd` or `imagick` for QR codes)
- Composer 2
- Node.js and npm (only if you rebuild frontend assets)
- MySQL 5.7+ or MariaDB 10.4+

## Setup

```bash
composer install
cp .env.example .env   # create .env yourself if this file is missing
php artisan key:generate
```

There is no `.env.example` in this repository. Create `.env` with at least:

```dotenv
APP_NAME=MidwayCafe
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=rms2
DB_USERNAME=root
DB_PASSWORD=

MAIL_MAILER=smtp
MAIL_HOST=127.0.0.1
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS=hello@example.com
MAIL_FROM_NAME="${APP_NAME}"

# SSLCommerz (sandbox by default)
API_DOMAIN_URL=https://sandbox.sslcommerz.com
STORE_ID=
STORE_PASSWORD=
IS_LOCALHOST=true
```

Import the bundled database. Migrations alone are not enough: `phone` and `salary` on `users`, menu category fields on `products`, and the `orders` table exist in the dump and not in every migration.

```bash
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS rms2 CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -p rms2 < rms2.sql
```

`rms2.sql` is a January 2023 phpMyAdmin dump. It includes sample customers, staff, menu items, carts, and reservations, plus real-looking names, emails, and phone numbers. Use it only on a local machine, and replace those rows before any shared or production database.

Start the app:

```bash
php artisan serve
```

Open `http://localhost:8000`. Sign-in and registration use Jetstream at `/login` and `/register`. After login, `/redirects` sends customers home and every other usertype to the admin dashboard.

Rebuild CSS and JS only if you change `resources/css`, `resources/js`, or Tailwind:

```bash
npm install
npm run dev
```

## Main routes

| Path | Who | What |
| --- | --- | --- |
| `/` | Public | Home |
| `/menu` | Public | Full menu |
| `/cart` | Customer | Cart |
| `/my-order` | Customer | Past orders |
| `/trace-my-order` | Public | Look up an invoice |
| `/top/rated` | Public | Highest-rated dishes |
| `/login`, `/register` | Guest | Jetstream auth |
| `/redirects` | Signed in | Role redirect |
| `/admin/home` | Admin | Dashboard |
| `/admin/food-menu` | Admin | Menu |
| `/admin/orders-incomplete` | Admin | New orders |
| `/orders/process` | Admin | Approved orders |
| `/orders-complete` | Admin | Delivered orders |
| `/orders/cancel` | Admin | Cancelled orders |
| `/admin/reservation` | Admin | Table bookings |
| `/admin/coupon`, `/admin/charge` | Admin | Discounts and fees |
| `/admin/banner/all`, `/admin/customize` | Admin | Homepage content |
| `/customer` | Admin | Customers |
| `/delivery-boy` | Admin | Delivery staff |

Payment callbacks live at `/pay`, `/success`, `/fail`, `/cancel`, `/ipn`, and under `/bkash/*`.

## Payments and mail

- **SSLCommerz** reads `config/sslcommerz.php`. Sandbox is the default `API_DOMAIN_URL`. Set `STORE_ID` and `STORE_PASSWORD` before checkout.
- **bKash** is wired for the sandbox checkout API. App key, secret, username, and password are hardcoded in `app/Http/Controllers/BkashController.php`. Move those to `.env` before using a real merchant account, and do not commit live credentials.
- Placed orders, approvals, reservations, and new staff accounts send mail through Laravel's mailer (`app/Mail`). Point `MAIL_*` at Mailhog, Mailpit, or a real SMTP server or those steps will fail after the order is saved.

Cash on delivery does not call a gateway. The cart is marked placed and a PDF receipt is generated in `ShipmentController`.

## Tests

```bash
php artisan test
```

The suite is the Jetstream default (registration, login, profile). It does not cover menu, cart, or admin flows.

## Notes

- `routes/web.php` applies a `customAuth` middleware to the bKash routes, but that alias is not registered in `app/Http/Kernel.php`.
- Admin authorization is an inline `usertype != 1` check inside controller methods, not route middleware.
- Product, chef, and banner images are served from `public/assets/images`.
