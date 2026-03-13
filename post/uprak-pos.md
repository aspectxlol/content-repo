---
title: "Building UPRAK-POS: A Terminal-Based POS System for My Ujian Praktik"
excerpt: "I had to build a cashier system for my final practical exam. Here's how far I took it."
coverImage: "/assets/blog/uprak-pos/cover.jpeg"
date: "2026-03-13T05:35:07.322Z"
---

# Building UPRAK-POS: A Terminal-Based POS System for My Ujian Praktik

I had to build a cashier system for my Ujian Praktik. The criteria: implement QR-based payment, and go as far as you can. Here's how far I went.

---

## Why a POS System?

The theme was already set — cashier/sales system — so the question wasn't *what* to build, it was *how far to take it*. A POS also happened to be personally meaningful. My family runs a small business and they track sales manually — like, pen and paper manually. So this wasn't just an exam project; it could actually be useful.

It also gave me a lot of surface area to work with on the technical side:

- **OOP** — a natural fit for modeling products, carts, and transactions
- **File I/O** — CSV for persistent storage, TXT for receipts
- **Input validation** — lots of it, across every user-facing input
- **Terminal UI** — ANSI color codes to make it not look terrible
- **QR payment** — required by the criteria, and genuinely interesting to implement

---

## The Architecture Decision: One Class to Rule Them All

Early on I had to decide how to structure the code. I went with a single `POS` class that holds all the state — `products` and `cart` as instance variables — and all the logic as methods.

```python
class POS:
    def __init__(self) -> None:
        self.products = []  # List of dicts: {id, name, price}
        self.cart = []      # List of dicts: {id, name, price, qty}
        self.next_product_id = 1
        self.products_file = "products.csv"
        self.load_products()
```

Simple, flat, all in one place. For a school project scope, this made sense. No need for separate `ProductManager` or `CartService` classes — that would've been over-engineering it.

The two core data structures are just **lists of dictionaries**:

```python
# Products loaded from CSV
products = [
    {"id": 1, "name": "Nasi Goreng", "price": 12000},
    {"id": 2, "name": "Teh Manis",   "price": 5000},
]

# Cart built during a session
cart = [
    {"id": 1, "name": "Nasi Goreng", "price": 12000, "qty": 2},
]
```

No database, no ORM, no external dependencies beyond `qrcode`. Just Python and a CSV file. That was a deliberate choice — keep the deployment story as simple as possible.

---

## Building It: The Flow

The application follows a simple dispatch loop:

```
run_application()  →  main_menu()  →  choice  →  method()  →  repeat
```

The `run_application()` function is basically a `while True` loop that renders the menu, captures input, and calls the right method. Dead simple, and it works.

The method breakdown ended up like this:

| Method | What it does |
|---|---|
| `load_products()` | Reads `products.csv` into `self.products` on startup |
| `save_products()` | Writes current state back to CSV after every change |
| `add_product()` | Prompts for name + price, appends to list, saves |
| `edit_product()` | Finds by ID, updates fields, saves |
| `add_to_cart()` | Finds product by ID, accumulates qty if already in cart |
| `remove_from_cart()` | Pops item by ID from cart |
| `checkout()` | Full payment flow — cash or QRIS — then clears cart |
| `generate_receipt()` | Writes a timestamped `.txt` file |
| `display_qr_code()` | Generates a scannable QR using `qrcode[pil]` |

---

## The Challenges

### 1. Cart accumulation logic

The cart wasn't supposed to add duplicate rows — if you add "Nasi Goreng" twice, it should just increment the `qty`. Getting this right took a couple of tries.

```python
cart_item = next((c for c in self.cart if c['id'] == pid), None)
if cart_item:
    cart_item['qty'] += qty
else:
    self.cart.append({...})
```

Using `next()` with a generator expression here is clean and readable. I liked how this turned out.

### 2. Input validation everywhere

Every single input needed to be validated. Numeric checks, empty string checks, ID existence checks, "is this cash amount enough" checks. I ended up writing two helper functions early on — `input_number()` and `input_int()` — that loop until they get valid input. That saved a lot of repeated `try/except` blocks throughout the methods.

### 3. The Colab vs. local problem

The project needed to run in **two environments**: locally in a terminal, and in Google Colab as a `.ipynb` notebook. This created a real pain point with `clear_screen()`.

Locally, `os.system('clear')` works fine. In Colab, it does nothing. The fix was to use IPython's `clear_output()`:

```python
def clear_screen() -> None:
    clear_output(wait=True)
    sys.stdout.flush()
    time.sleep(0.5)
```

The `time.sleep(0.5)` was also needed because Colab's input/output pipeline has a slight async delay — without it, inputs would sometimes fire before the screen re-rendered, which looked broken.

### 4. QRIS payment

Most people probably slapped a static QR image on screen and called it done. I wanted something that actually made sense as a system.

The QR code I generate encodes real payment data — customer name, merchant name, and the transaction amount — serialized as JSON, base64-encoded, and appended to a live URL I hosted myself:

```python
payment_data = {
    "merchantName": "Ujian Praktek",
    "customerName": customer_name,
    "price": str(int(amount))
}
encoded_data = base64.b64encode(json.dumps(payment_data).encode()).decode()
payment_url = f"https://louie.is-a.dev/random/uprak-pos/payment?data={encoded_data}"
```

Every QR code generated is unique to that transaction. Scan it and you land on an actual page that decodes the URL and shows you the payment details. Not a real payment gateway — but a real system. No two transactions produce the same code, and the data travels with the QR itself rather than being stored server-side. That felt like the right way to do it.

---

## What I Learned

**OOP is most useful when state and behavior belong together.** The `POS` class isn't just a namespace for functions — `self.products` and `self.cart` being instance variables means every method naturally has access to the current state. It clicked for me why OOP exists.

**No database is a valid choice for the right scope.** CSV files are readable, editable in Excel, and require zero setup. For a local, single-user POS that a canteen worker uses, this is genuinely fine. It becomes a problem if you need multi-user access or sales history queries — but that wasn't the requirement here.

**Validation is half the work.** When I first wrote `add_product()`, I skipped most validation. Then I tested it with bad inputs and everything broke in annoying ways. Going back and adding proper validation to every input point took almost as long as writing the features themselves.

**Target environment matters early.** I almost finished the whole thing before realizing the Colab version needed different screen-clearing logic. That was an annoying thing to retrofit. Next time I'd think about deployment context from day one.

---

## What I'd Change

Looking back, a few things I'd do differently:

- **Stock tracking.** Right now there's zero inventory management — you can sell unlimited quantities of anything. Even a simple "stock" field on the product would've made this more realistic.
- **Sales history.** Receipts are raw `.txt` files. A simple CSV sales log would've made it possible to query past transactions without reading individual receipt files.
- **Separate the Colab version properly.** The `.ipynb` patches methods onto the class after the fact (`POS.add_product = add_product`), which works but is a bit hacky. If I were redoing this I'd structure the notebook differently from the start.

---

## Shoutouts

This was technically a solo coding project, but I didn't do it completely alone. Big thanks to **Destiana, Echa, and Yosua** — my UPRAK project mates — who handled the slides and presentation deck, ran bug testing, and threw around ideas I actually ended up using. The terminal color scheme came out of one of those conversations. The project would've looked and felt worse without them.

---

## Final Thoughts

UPRAK-POS ended up being one of the more complete things I've shipped for school. It runs, it handles bad input, it generates real receipts, it has a QR code that actually encodes transaction data — it works.

The goal was to demonstrate Python concepts for an exam, but it also ended up being something that could genuinely be used at a small canteen with minimal setup. That felt good.

If you want to try it out: run `pip install qrcode[pil]`, grab `main.py`, and run it. The first time you launch it there are no products — add a few with option `1`, then try checking out.

---

*Built for UPRAK (Ujian Praktik) — by Louie Hansen Linadi*