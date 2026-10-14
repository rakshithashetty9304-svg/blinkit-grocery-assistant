# Blinkit Grocery Assistant — Power Automate Desktop

A Power Automate Desktop (PAD) bot that does a weekly grocery run on **blinkit.com** up to the cart, then **stops before checkout and payment**. It reads a configurable shopping list, verifies the delivery address, compares products, builds and checks the cart against a budget, writes a shopping summary, and sends that summary by **Gmail** and **WhatsApp**.

> No order is placed and no payment is started. The bot only prepares the cart.

Built end to end with AI-assisted development (Claude Code), with every subflow reviewed and tested on live runs. Personal details (address, phone, email) are hidden in this public copy.

---

## 1. What the bot does

| Step | What happens | Subflow |
|---|---|---|
| 1 | Reads `config/settings.csv` (budget, address keyword, recipients, output folder) and `config/items.csv` (shopping list) | `Main` |
| 2 | Opens Blinkit in Microsoft Edge and **verifies the delivery address** contains the configured keyword. If it doesn't, the bot **sets it**: it opens the location picker, selects the saved address that contains the keyword, and verifies again. If it still doesn't match, it stops safely. | `ShoppingProcess` |
| 3 | **Empties the cart** so every run starts clean (with a fallback if Blinkit's cart panel fails) | `ShoppingProcess` |
| 4 | For each item: searches Blinkit, reads every product card (name, size, price, stock), **compares products** and picks the best one, adds it, and raises the quantity | `ShoppingProcess` |
| 5 | **Verifies each quantity** in the cart, then **checks the cart total against the budget** | `ShoppingProcess` |
| 6 | Writes `output/shopping_summary.txt` | `Main` |
| 7 | Emails the summary, with the file attached, through Gmail SMTP | `SendEmail` |
| 8 | Sends the summary on WhatsApp through WhatsApp Web | `SendWhatsApp` |
| — | Any failure is logged to `output/error_log.txt` and listed in the summary under **Issues found during run** | `HandleError` |

### How products are compared
For each item the bot keeps only products that:
1. have a name containing one of the `MustContain` keywords (so a "milk" search ignores banana chips),
2. are **in stock**, meaning they show an **ADD** button,
3. cost no more than `MaxUnitPrice`.

From those, it picks the **best value: the lowest price per 100 g or 100 ml**. If a product's size can't be read, it falls back to the lowest price. This avoids choosing a tiny pack just because it's cheapest.

### Built-in checks (error handling)
| Situation | Bot behaviour |
|---|---|
| Delivery address doesn't match the keyword | Switches to the saved address containing the keyword, then re-verifies. If it still doesn't match, it stops before shopping and logs the error. |
| Cart can't be emptied (for example, items not on the list) | Stops before shopping, logs the error |
| No in-stock product matches within the max price | Skips that item, logs the reason |
| Blinkit won't allow the full quantity (limited stock or per-order limit) | Keeps what it could add, reports `SHORT, wanted X but cart has Y` |
| Cart total exceeds the budget | Summary status `OVER BUDGET` |
| WhatsApp not logged in, invalid number, or chat didn't load | Retries once, then logs the error |

Blinkit is a live store and stock changes during the day. The bot reads what's actually available on each run, so the chosen product can differ between runs. The quantity check reports any shortfall.

---

## 2. Project structure

```
blinkit-grocery-assistant/
├── README.md                  ← this file (setup + design)
├── config/
│   ├── items.csv              ← shopping list (edit this, not the flow)
│   └── settings.csv           ← budget, address keyword, recipients, output folder
├── flows/                     ← PAD flow source (copy-paste into PAD)
│   ├── 1_HandleError.txt
│   ├── 2_ShoppingProcess.txt
│   ├── 3_Main.txt
│   ├── 4_SendWhatsApp.txt
│   └── 5_SendEmail.txt        ← App password removed: enter your own
├── output/
│   ├── shopping_summary.txt   ← written by the bot on every run
│   └── error_log.txt          ← appended by HandleError
└── screenshots/               ← evidence of a full run (PAD flow, cart, Gmail, WhatsApp, output)
```

---

## 3. Configuration

### `config/items.csv`
| Column | Meaning | Example |
|---|---|---|
| `Item` | Name used in the summary | `Milk` |
| `SearchTerm` | Text typed into Blinkit search | `toned milk` |
| `MustContain` | Words the product name must contain, separated by `;` | `curd;dahi;yogurt` |
| `Quantity` | Units to add | `2` |
| `MaxUnitPrice` | Highest acceptable price per unit (₹) | `40` |

### `config/settings.csv`
Keep the rows in this order, because the flow reads them by position.

| Key | Meaning |
|---|---|
| `Budget` | Maximum cart item total (₹) |
| `DeliveryAddressKeyword` | A word that must appear in Blinkit's delivery address, such as your area name |
| `EmailRecipient` | Test email address that receives the summary |
| `WhatsAppRecipient` | Test WhatsApp number: country code + number, digits only (e.g. `91XXXXXXXXXX`) |
| `OutputFolder` | Folder for `shopping_summary.txt` and `error_log.txt` |

---

## 4. Setup instructions

**Requirements:** Windows 10/11, Power Automate Desktop (free), Microsoft Edge, a Blinkit account, a Gmail account with 2-Step Verification, and WhatsApp on a phone.

1. **Copy the project folder** to `D:\SONY\blinkit-pad-assignment`. If you use a different location, update the two file paths at the top of `flows/3_Main.txt` and the `OutputFolder` row in `settings.csv`.
2. **Install the PAD browser extension for Edge.** In PAD, go to **Tools → Browser extensions → Microsoft Edge** and turn it on.
3. **Sign in once in Edge (manual, one time):**
   - **blinkit.com**: log in with OTP and choose your delivery address.
   - **web.whatsapp.com**: scan the QR code from your phone (WhatsApp → Linked devices).
4. **Create a Gmail App password**: Google Account → Security → 2-Step Verification ON → App passwords. Copy the 16-character password.
5. **Edit `config/settings.csv`** with your address keyword, test email and test WhatsApp number.
6. **Create the flow in PAD:**
   1. **New flow** → name it `Blinkit Grocery Assistant`, with **Power Fx off**.
   2. Create 4 subflows: `HandleError`, `ShoppingProcess`, `SendEmail`, `SendWhatsApp`.
   3. Paste each file from `flows/` into its subflow: open the file in Notepad, press Ctrl+A and Ctrl+C, then press Ctrl+V in the subflow. Paste `3_Main.txt` into **Main** last.
   4. In **SendEmail**, open the *Send email* action and enter your Gmail address and App password.
7. **Run** (▶). Don't use the mouse or keyboard during the run (about 2–3 minutes).

### Notes and assumptions
- The Blinkit login and the WhatsApp QR scan are done manually once. The bot never types OTPs or passwords into Blinkit or WhatsApp.
- Budget is compared with the **cart item total**. Delivery and handling charges are added later by Blinkit at checkout, which the bot deliberately doesn't open.
- Product data is read through PAD's *Run JavaScript function on web page* action. This is more reliable than screen coordinates and still works when the product list changes.
- Notifications go only to your own accounts and approved test recipients.

---

## 5. Sample output

```
BLINKIT GROCERY ASSISTANT - SHOPPING SUMMARY
Run time: 10/13/2026 9:12:24 PM
Delivery address: Delivery in 8 minutes Home ...

Items:
Milk: Heritage Daily Health Toned Milk (500 ml) at Rs 28 x 2 | 9 options compared, best value Rs 5.6 per 100 g/ml | quantity check: OK
Bread: Britannia Milk Bread (400 g) at Rs 55 x 1 | 9 options compared, best value Rs 13.8 per 100 g/ml | quantity check: OK
Yogurt: Nandini Pouch Curd (500 g) at Rs 28 x 1 | 8 options compared, best value Rs 5.6 per 100 g/ml | quantity check: OK

Cart total: Rs 139 | Budget: Rs 250 | Status: WITHIN BUDGET

Issues found during run: 0

Order NOT placed. The bot stops at the cart, before checkout and payment.
```
