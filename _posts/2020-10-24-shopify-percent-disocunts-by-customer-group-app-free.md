---
layout: post
title: "Shopify Plus: App Free % Based Customer Group Pricing"
date: 2020-10-24 00:00:00 +1300
tags: [Shopify, Shopify Plus]
---

### Background

When migrating a store from SuiteCommerce (SCA) to Shopify Plus I needed to recreate SCA's percentage based customer price level functionality. This set up followed a fairly common pattern, whereby select groups of customers receive a fixed percentage off RRP pricing, at various tiers (30% Off, 40% Off, think VIP, GOLD, Silver etc.).

There are many app based solutions out there aimed at wholesale or VIP tiered pricing, but I inherently don't want to use an app for anything unless the overhead (and cost) is 100% justified. In this case some minimal frontend JS + Liquid and checkout scripts can achieve the goal without too much heavy lifting.

### The Pieces:

1.  Set up your customer tags
2.  Make additions/changes in Liquid templates
3.  Make additions change to your themes JS files
4.  Add your cart script
5.  Add some other relevant script logic

## 1. Set up tagging

The easy bit! For the sake of this demo I will use some common examples:

Wholesale - 50% off RRP

VIP - 25% off RRP

Friends & Family - 10% off RRP

To keep things a little tidier in the admin we'll prefix the customer tags so we have:

`price_group:Wholsale`

`price_group:VIP`

`price_group:Friends Family`

A few points worth noting:

- Following this prefix: pattern is good practice as in other contexts you may want to leverage this key:value syntax (it also more informative to your admin users).
- Notice I drop the '&' from 'Friends & Family' in the tag name - Shopify is very good at sanitizing things under the hood, but adding unecessary special characters into your world should always be avoided.
- This is a perfectly valid use case for customer metafields (in fact the production version of this code uses a metafield for price tier), however if a store is not already utilising metafields (and an associated metafield management app) then it's not a necessary additional step/complexity: tags work just fine for simple use cases, and are more easily used for integrations (say, email segments based on your tiers).

## 2. Liquid template editing

### Global - layout/theme.liquid

The first thing we will do is to is include a snippet in the head of the theme.liquid to output some Liquid variables / JS const globally that will be used to calculate pricing. The below goes somewhere near the closing `</head>` tag.

```liquid
{% raw %}{% assign is_cg_account = false %}
{% assign cg_discount_multiplier = 0 %}
  {% if customer %}
    {% for tag in customer.tags %}
    {% assign tag_split = tag | split: ":" %}
    {% if tag_split[0] == "price_group" %}
      {% assign price_group = tag_split[1] %}
      {% if price_group == "Wholesale" %}
        {% assign is_cg_account = true %}
        {% assign cg_discount_multiplier = 0.5 %}
      {% elsif price_group == "VIP" %}
        {% assign is_cg_account = true %}
        {% assign cg_discount_multiplier = 0.75 %}
      {% elsif price_group == "Friends Family" %}
        {% assign is_cg_account = true %}
        {% assign cg_discount_multiplier = 0.9 %}
      {% endif %}
    {% endif %}
  {% endfor %}
{% endif %}
<script>
var is_cg_account = "{{is_cg_account | json}}"
var cg_discount_multiplier = "{{cg_discount_multiplier | json}}";
</script>{% endraw %}
```
Things to note:
- The multipliers are 'inverted' for simplicty; because 100\*0.75 is easier than 100-(100\*0.25).
- The `| json` filter here is a shorthand way to reliably pass liquid objects and variables into JS. Even though we are only passing one boolean and one decimal integer in this case, it's good practice to always use `| json` when surfacing anything from Liquid to frontend JS.


*The following Liquid template updates are not strictly necessary on many themes: your pricing display is likely oveidden with JS on the frontend anyway. But as an insurance against flashes of different prices etc. you can render the discounted price in Liquid first. Also the logic is the same in both the Liquid and frontend JS rendering of discounts,but the Liquid version is more readable, so a good intro.*

### PDP - sections/product-template.liquid

This will vary a little theme by theme, but you will have something that looks like this in your product-template.liquid file (or a snippet rendered within that file). This example is from the Turbo theme by Out of the Sandbox.

```liquid
{% raw %}{% if variant.price < variant.compare_at_price and variant.available %}
<span class="was_price">
  <span class="money">{{ variant.compare_at_price | money }}</span>
</span>
{% endif %}
<span itemprop="price" content="{{ variant.price | money_without_currency | remove: " ," }}" class="{% if variant.compare_at_price > variant.price %}sale{% endif %}">
  <span class="current_price {% if product.available == false %}hidden{% endif %}">
  {% if variant.price > 0 %}
    <span class="money">{{ variant.price | money }}</span>
  {% else %}
    {{ settings.free_price_text }}
  {% endif %}
    </span>
  </span>
  {% if section.settings.display_savings %}
  <p>
    <span class="sale savings">
    {% if variant.price < variant.compare_at_price and variant.available %} 
      {{ 'products.product.savings' | t }} {{ variant.compare_at_price | minus: variant.price | times: 100 | divided_by: variant.compare_at_price }}% (<span class="money">{{ variant.compare_at_price | minus: variant.price | money }}</span>)
    {% endif %}
    </span>
  </p>
{% endif %}{% endraw %}
```
This get replaced with the below:

```liquid
{% raw %}{% assign final_price = variant.price %}
{% if is_cg_account %}
  {% assign cg_discount_price = variant.compare_at_price | times: cg_discount_multiplier %}
  {% if cg_discount_price < variant.price %}
    {% assign final_price = cg_discount_price %}
  {% endif %}
{% endif %}
{% if final_price < variant.compare_at_price and variant.available %}
<span class="was_price">
  <span class="money">{{ variant.compare_at_price | money }}</span>
</span>
{% endif %}
<span itemprop="price" content="{{ final_price | money_without_currency | remove: " ," }}" class="{% if variant.compare_at_price > final_price %}sale{% endif %}">
  <span class="current_price {% if product.available == false %}hidden{% endif %}">
  {% if final_price > 0 %}
    <span class="money">{{ final_price | money }}</span>
  {% else %}
    {{ settings.free_price_text }}
  {% endif %}
    </span>
  </span>
  {% if section.settings.display_savings %}
  <p>
    <span class="sale savings">
    {% if final_price < variant.compare_at_price and variant.available %} 
      {{ 'products.product.savings' | t }} {{ variant.compare_at_price | minus: final_price | times: 100 | divided_by: variant.compare_at_price }}% (<span class="money">{{ variant.compare_at_price | minus: final_price | money }}</span>)
    {% endif %}
    </span>
  </p>
{% endif %}{% endraw %}
```
Things to note:
- In essence we capture the product price in our own temp variable, then discount that price based on the global parameters we added in the first step if they are present.
- We allow for the possibilty of the public on site price being lower than the customer group discount off RRP, and always use the lower price: `if cg_discount_price < variant.price`

<script src="https://gist.github.com/stevenhoney/0c9dc2721f6e7bcbb1d61f0ffc295ad0/revisions?diff=split"></script>

