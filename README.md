# Fixing Shopify India GST Inversion: Step-by-Step Guide for Store Owners

If you run an Indian Shopify store, chances are you’ve printed an invoice using Order Printer only to find a bizarre tax breakdown: a customer in your home state was charged IGST, while an out-of-state customer received an intrastate split of CGST + SGST.

> **Note on Tax Slabs:** This guide is specifically designed for stores operating with a **flat 18% GST** across their catalog. If your store sells products across **mixed GST slabs** (such as 5% on wooden toys/handicrafts or value-based apparel where items ≤ ₹1,000 are taxed at 5% and items > ₹1,000 are taxed at 12%), please refer to the advanced multi-rate guide and template in this repository: [shopify-india-multislab-gst](https://github.com/sandeepsingh-tech/shopify-india-multislab-gst).

Here is how this happens, why Shopify behaves this way, and how to fix it across your admin settings and Order Printer templates.

---

## 1. How Store Owners End Up in This Mess

Shopify powers thousands of Indian e-commerce merchants, but its default tax configuration is not plug-and-play for Indian GST rules:

* **The Karnataka Default Trap:** When you activate India under Shopify Taxes, the template assumes the business is operating from **Karnataka**. It pre-configures Karnataka to charge `SGST added to federal tax (CGST)` and defaults all remaining states (Gujarat, Maharashtra, Delhi, etc.) to `IGST instead of federal tax`.


* **Incomplete Shipping Origin:** During initial store setup, many merchants leave their default fulfillment location set with just `India`—missing the exact street address, city, state, and PIN code. Without an origin state, Shopify's tax engine cannot confirm where an order ships from.


* **Frozen Checkout Records:** Shopify writes tax lines to an order at the time of checkout. Once an order is placed, editing admin tax settings will **not** retroactively recalculate older orders. Order Printer will keep pulling the old, incorrect tax entries stored in that order's data.



---

## 2. Step-by-Step Fix in Shopify Admin

Fixing this in Shopify Admin ensures that **all new and upcoming checkouts** calculate taxes properly.

### Step 1: Assign Your True Store Location

1. In Shopify Admin, open **Settings > Locations**.


2. Click your primary dispatch/warehouse location and confirm that the **Address**, **City**, **Postal code**, and **State / Province** are fully filled out.


3. Under **Default location**, verify that this specific warehouse is selected as the default origin.



### Step 2: Correct the Base Taxes for Your State

1. Open **Settings > Taxes and duties > India**.


2. Under **Base taxes**, check the top federal field:
* **India (Federal tax):** Set to `9 %` (this covers central CGST for standard 18% products).




3. Scroll through the state list and update three specific entries:


* **Your Home State (e.g., Gujarat / Maharashtra / Delhi):**
* Rate: `9 %`
* Label: `SGST`
* Dropdown: Change to **`added to 9% federal tax`**


* **Karnataka (Remove Shopify Default):**
* Rate: `18 %`
* Label: `IGST`
* Dropdown: Change to **`instead of 9% federal tax`**


* **All Other States:**
* Confirm every other state/UT is set to `18 %` | `IGST` | **`instead of 9% federal tax`**.






4. Click **Save**.



---

## 3. Resolving Historical Invoices in Order Printer

Because past orders keep their historical tax data, replacing the rigid `order.tax_lines` loop in your **Order Printer** template with a dynamic cutoff script will recalculate historical orders while letting future orders run natively.

### How the Hybrid Template Operates

* **Orders on or before your fix date:** Reads the total tax collected and dynamically splits it into **CGST + SGST** (if shipped to your state) or **IGST** (if shipped elsewhere), fixing past mistakes without disturbing order subtotals or total amounts paid.
* **Orders after your fix date:** Seamlessly switches back to Shopify's native `order.tax_lines` loop so product overrides (5%, 12%, 28%) and new line items are handled as configured.

---

## 4. The Complete Universal Order Printer Template

Copy and paste this template directly into your Shopify Order Printer app (**Apps > Order Printer > Templates > Invoice**):

```html
<div>
  <!-- ======================================================== -->
  <!-- 1. HEADER: BRAND LOGO & TAX INVOICE METADATA            -->
  <!-- ======================================================== -->
  <div class="columns" style="align-items: center; border-bottom: 2px solid #000; padding-bottom: 12px; margin-bottom: 16px;">
    <div>
      <!-- CUSTOMIZATION: Add an <img> tag with your store logo URL if needed -->
      <h1 style="margin: 0; font-size: 22px; font-weight: bold; text-transform: uppercase;">{{ shop.name }}</h1>
      <h2 style="margin: 4px 0 0 0; font-size: 15px; text-transform: uppercase; letter-spacing: 0.5px; color: #444;">TAX INVOICE</h2>
    </div>
    <div>
      <p style="text-align: right; margin: 0; line-height: 1.45; font-size: 13px;">
        <strong>Invoice No:</strong> {{ order.order_name }}<br />
        <strong>Invoice Date:</strong> {{ order.created_at | date: "%d %b %Y" }}<br />
        {% if order.po_number %}<strong>PO No:</strong> {{ order.po_number }}<br />{% endif %}
        {% if order.shipping_address %}
          <strong>Place of Supply:</strong> {{ order.shipping_address.province }} ({{ order.shipping_address.province_code }})
        {% endif %}
      </p>
    </div>
  </div>

  <!-- ======================================================== -->
  <!-- 2. ADDRESS GRID: SELLER, BILLED TO, SHIPPED TO           -->
  <!-- ======================================================== -->
  <div class="columns" style="margin-top: 1em; line-height: 1.45; font-size: 13px;">
    <!-- Seller / From Details -->
    <div class="address" style="flex: 1; padding-right: 15px;">
      <strong style="text-transform: uppercase; font-size: 12px; color: #555;">Sold By (From)</strong><br/>
      <strong style="font-size: 14px;">{{ shop.name }}</strong><br/>
      {{ shop.address | format_address }}
      {% if shop.phone %}Tel: {{ shop.phone }}<br/>{% endif %}

      <!-- CUSTOMIZATION: Replace placeholders with your registered GSTIN, State Name, and Code -->
      <strong style="display: inline-block; margin-top: 4px; color: #111;">GSTIN: [YOUR_GSTIN_HERE]</strong><br/>
      <span>State: [YOUR_STATE_NAME] (Code: [STATE_CODE])</span>
    </div>

    <!-- Billed To -->
    {% if order.billing_address %}
    <div class="address" style="flex: 1; padding-right: 15px;">
      <strong style="text-transform: uppercase; font-size: 12px; color: #555;">Billed To</strong>
      {{ order.billing_address | format_address }}
      {% if order.billing_address.phone %}Tel: {{ order.billing_address.phone }}<br/>{% endif %}
      {% if order.note_attributes.gstin %}
        <strong>Customer GSTIN:</strong> {{ order.note_attributes.gstin }}<br/>
      {% endif %}
    </div>
    {% endif %}

    <!-- Shipped To -->
    {% if order.shipping_address %}
    <div class="address" style="flex: 1;">
      <strong style="text-transform: uppercase; font-size: 12px; color: #555;">Shipped To</strong>
      {{ order.shipping_address | format_address }}
      {% if order.shipping_address.phone %}Tel: {{ order.shipping_address.phone }}{% endif %}
    </div>
    {% endif %}
  </div>

  <hr style="margin: 18px 0 12px 0; border: none; border-top: 1px solid #ddd;" />

  <!-- ======================================================== -->
  <!-- 3. ORDER ITEMS TABLE                                     -->
  <!-- ======================================================== -->
  <h2 style="font-size: 15px; margin-bottom: 8px;">Order Details</h2>
  <table class="table-tabular" style="width: 100%; border-collapse: collapse; margin-top: 6px;">
    <thead>
      <tr style="border-bottom: 2px solid #333; text-align: left;">
        <th style="width: 8%; padding: 6px 4px;">Qty</th>
        <th style="padding: 6px 8px;">Item Description</th>
        <th style="width: 14%; padding: 6px 8px;">SKU / HSN</th>
        <th style="width: 18%; text-align: right; padding: 6px 4px;">Price</th>
      </tr>
    </thead>
    <tbody>
      {% for line_item in order.line_items %}
      <tr style="border-bottom: 1px solid #eee;">
        <td style="padding: 8px 4px; vertical-align: top;">{{ line_item.quantity }}</td>
        <td style="padding: 8px 8px; vertical-align: top;">
          <strong>{{ line_item.title }}</strong>
          {% if line_item.line_level_discount_allocations.size > 0 %}
            <div class="subduedText" style="font-size: 12px; color: #666;">
              {% for discount_allocation in line_item.line_level_discount_allocations %}
                Discount: {{ discount_allocation.discount_application.title }} (-{{ discount_allocation.amount | money }})<br>
              {% endfor %}
            </div>
          {% endif %}
        </td>
        <td style="padding: 8px 8px; vertical-align: top; color: #555; font-size: 12px;">
          {{ line_item.sku | default: "—" }}
        </td>
        <td style="text-align: right; padding: 8px 4px; vertical-align: top;">
          {% if line_item.original_price != line_item.final_price %}
            <span class="subduedText"><s style="color: #888;">{{ line_item.original_price | money }}</s></span><br/>
          {% endif %}
          <strong>{{ line_item.final_price | money }}</strong>
        </td>
      </tr>
      {% endfor %}

      <!-- Subtotal -->
      <tr>
        <td colspan="3" style="text-align: right; padding: 6px 8px; font-weight: bold;">Subtotal</td>
        <td style="text-align: right; padding: 6px 4px;">{{ order.line_items_subtotal_price | money }}</td>
      </tr>

      <!-- Discounts -->
      {% assign has_shipping_discount = false %}
      {% for discount_application in order.discount_applications %}
        {% if discount_application.target_type == 'shipping_line' %}
          {% assign has_shipping_discount = true %}
        {% endif %}
        {% if discount_application.target_selection == 'all' and discount_application.target_type != 'shipping_line' %}
        <tr>
          <td colspan="3" style="text-align: right; padding: 4px 8px; color: #555;">Discount ({{ discount_application.title }})</td>
          <td style="text-align: right; padding: 4px 4px;">-{{ discount_application.total_allocated_amount | money }}</td>
        </tr>
        {% endif %}
      {% endfor %}

      <!-- ======================================================== -->
      <!-- 4. HYBRID GST TAX CALCULATION (HISTORICAL + FUTURE)       -->
      <!-- ======================================================== -->
      {% if order.tax_price > 0 %}
        <!-- CUSTOMIZATION: Set your home state 2-letter code (e.g. GJ, MH, DL, KA) -->
        {% assign seller_state_code = 'GJ' %}

        <!-- CUSTOMIZATION: Date you updated base taxes in Shopify Admin (YYYY-MM-DD) -->
        {% assign settings_fix_date = '2026-11-04' %}
        {% assign order_date = order.created_at | date: '%Y-%m-%d' %}

        {% if order_date <= settings_fix_date %}
          <!-- MODE A: HISTORICAL ORDER OVERRIDE -->
          {% assign is_intrastate = false %}
          {% if order.shipping_address and order.shipping_address.province_code == seller_state_code %}
            {% assign is_intrastate = true %}
          {% elsif order.shipping_address == blank and order.billing_address.province_code == seller_state_code %}
            {% assign is_intrastate = true %}
          {% endif %}

          {% if is_intrastate %}
            <!-- Intra-state Supply: Splits total tax equally into CGST (9%) and SGST (9%) -->
            {% assign half_tax = order.tax_price | divided_by: 2.0 %}
            <tr>
              <td colspan="3" style="text-align: right; padding: 4px 8px; color: #444;">CGST (9.0%)</td>
              <td style="text-align: right; padding: 4px 4px;">{{ half_tax | money }}</td>
            </tr>
            <tr>
              <td colspan="3" style="text-align: right; padding: 4px 8px; color: #444;">SGST (9.0%)</td>
              <td style="text-align: right; padding: 4px 4px;">{{ half_tax | money }}</td>
            </tr>
          {% else %}
            <!-- Inter-state Supply: Displays total tax under IGST (18%) -->
            <tr>
              <td colspan="3" style="text-align: right; padding: 4px 8px; color: #444;">IGST (18.0%)</td>
              <td style="text-align: right; padding: 4px 4px;">{{ order.tax_price | money }}</td>
            </tr>
          {% endif %}

        {% else %}
          <!-- MODE B: NATIVE TAX ENGINE (For all future orders) -->
          {% for tax_line in order.tax_lines %}
          <tr>
            <td colspan="3" style="text-align: right; padding: 4px 8px; color: #444;">
              {{ tax_line.title }} ({{ tax_line.rate | times: 100 }}%)
            </td>
            <td style="text-align: right; padding: 4px 4px;">{{ tax_line.price | money }}</td>
          </tr>
          {% else %}
          <tr>
            <td colspan="3" style="text-align: right; padding: 4px 8px;">Taxes (Included)</td>
            <td style="text-align: right; padding: 4px 4px;">{{ order.tax_price | money }}</td>
          </tr>
          {% endfor %}
        {% endif %}

      {% else %}
        <tr>
          <td colspan="3" style="text-align: right; padding: 4px 8px; color: #444;">Taxes</td>
          <td style="text-align: right; padding: 4px 4px;">{{ 0 | money }}</td>
        </tr>
      {% endif %}

      <!-- Shipping -->
      {% if order.shipping_address %}
      <tr>
        <td colspan="3" style="text-align: right; padding: 4px 8px;">Shipping</td>
        <td style="text-align: right; padding: 4px 4px;">
          {% if has_shipping_discount %}FREE{% else %}{{ order.shipping_price | money }}{% endif %}
        </td>
      </tr>
      {% endif %}

      <!-- Total -->
      <tr style="border-top: 2px solid #333; border-bottom: 2px solid #333; font-size: 15px;">
        <td colspan="3" style="text-align: right; padding: 8px 8px;"><strong>Invoice Total</strong></td>
        <td style="text-align: right; padding: 8px 4px;"><strong>{{ order.total_price | money }}</strong></td>
      </tr>
    </tbody>
  </table>

  <!-- ======================================================== -->
  <!-- 5. TERMS & SIGNATORY SECTION                             -->
  <!-- ======================================================== -->
  <div class="columns" style="margin-top: 28px; align-items: flex-end;">
    <div style="flex: 2; font-size: 11px; color: #555; line-height: 1.4;">
      <p style="margin: 0;"><strong>Terms & Conditions:</strong></p>
      <!-- CUSTOMIZATION: Modify or add your return/warranty policy bullets -->
      <ul style="padding-left: 14px; margin: 4px 0 0 0;">
        <li>Goods sold are covered under the merchant's replacement and return policy.</li>
        <li>This is a computer-generated tax invoice and requires no physical signature.</li>
      </ul>
    </div>
    <div style="flex: 1; text-align: right; font-size: 12px;">
      <p style="margin: 0 0 32px 0;"><strong>For {{ shop.name }}:</strong></p>
      <p style="margin: 0; border-top: 1px dashed #666; display: inline-block; padding-top: 4px;">Authorized Signatory</p>
    </div>
  </div>

  <!-- ======================================================== -->
  <!-- 6. FOOTER: SUPPORT EMAIL & STORE DOMAIN                  -->
  <!-- ======================================================== -->
  <hr style="margin: 18px 0 10px 0; border: none; border-top: 1px solid #ddd;" />
  <p style="text-align: center; margin: 0; font-size: 12px; color: #444;">
    {{ "invoice_template.email_footer" | t: email: shop.email }} | Visit us at <strong>{{ shop.domain }}</strong>
  </p>
</div>

```
