# Budds i18n, Markets, Automation, Debug

## Localization setup

Theme strings live in `locales/en.default.json` and `locales/uk.json`. The CSV source is `locales/budds-translations.csv`; it can be used as the working table for translators or an AI translation flow, then converted back to Shopify locale JSON.

The product section renders all user-facing copy through translation keys:

```liquid
{{ 'sections.budds_product_hero.heading' | t }}
{{ 'sections.budds_product_hero.shipping_text_html' | t: amount: free_shipping_amount }}
```

The section schema stores translation keys, not final text. This keeps the section editable in the theme editor while allowing the same section to render different languages.

Language switching uses Shopify's native localization form:

```liquid
{% form 'localization' %}
  <select name="locale_code" onchange="this.form.submit()">
    ...
  </select>
{% endform %}
```

This preserves Shopify's localized routing and avoids hand-built language URLs.

## Market metafields

Create market metafield definitions:

| Owner | Namespace | Key | Type | Example US | Example EU |
| --- | --- | --- | --- | --- | --- |
| Market | `budds` | `free_shipping_threshold` | Single line text | `$50` | `€45` |
| Market | `budds` | `hero_price` | Single line text | `$25` | `€23` |

Liquid reads them from the current market:

```liquid
{% liquid
  assign free_shipping_amount = localization.market.metafields.budds.free_shipping_threshold.value
  assign market_price = localization.market.metafields.budds.hero_price.value
%}
```

The shipping text is interpolated, so no amount is hardcoded in the theme:

```liquid
{{ 'sections.budds_product_hero.shipping_text_html' | t: amount: free_shipping_amount }}
```

## Market logic

The section detects US/EU from `localization.market.handle` and `localization.country.iso_code`. It changes:

- button URL: default / US / EU block settings;
- button translation key: default / US / EU;
- displayed price: `localization.market.metafields.budds.hero_price`;
- free shipping threshold: `localization.market.metafields.budds.free_shipping_threshold`.

Product pricing in production should still be configured in Shopify Markets price lists or catalog pricing. The section price is presentation copy only.

## AI translation automation flow

```text
1. Extract source strings
   - Read locale JSON or CSV source.
   - Keep key, source language, target languages, and context.

2. Send to AI
   - Prompt: translate only values, preserve keys, HTML tags, and Liquid placeholders.
   - Require valid JSON or CSV output.

3. Validate output
   - Check all source keys exist in target language.
   - Check placeholders match exactly: {{ amount }} stays {{ amount }}.
   - Check HTML tags are balanced.

4. Save result
   - Write target `locales/{locale}.json`, or update a translation spreadsheet.
   - Optional: push translations to Shopify with Admin API / Translation API.

5. Review and deploy
   - Human review for brand tone.
   - Run theme check and preview language routes.
```

Pseudo-code:

```js
const rows = readCsv('locales/budds-translations.csv');
const untranslated = rows.filter(row => !row.uk);

const translated = await ai.translate({
  sourceLocale: 'en',
  targetLocale: 'uk',
  rows: untranslated,
  rules: [
    'Preserve keys',
    'Preserve HTML',
    'Preserve Liquid placeholders like {{ amount }}'
  ]
});

validatePlaceholders(translated, untranslated);
writeLocaleJson('locales/uk.json', translated);
```

n8n version:

```text
Cron/Webhook
-> Google Sheets: Read rows
-> Claude/OpenAI: Translate missing target cells
-> Code node: validate placeholders + HTML
-> Google Sheets: Update rows
-> GitHub/Shopify Admin API: save locale JSON or translations
-> Slack: send review summary
```

## Debug: 404 or broken links after language change

Possible causes:

- language is enabled in theme files but not published/enabled in Shopify Markets;
- URL was built manually instead of using `{% form 'localization' %}`;
- product/page/blog handle is translated or missing in the target locale;
- app proxy, custom route, or redirect ignores locale prefixes;
- canonical/hreflang tags point to the wrong locale URL;
- market-specific domain/subfolder configuration changed;
- cached menu links still point to old handles.

Diagnosis process:

- Reproduce in Shopify preview with `?preview_theme_id=...` and each locale/market.
- Check Network tab for 301/302 chains, final URL, and status code.
- Inspect generated links in the DOM before clicking.
- Confirm `localization.language.iso_code`, `localization.market.handle`, and `request.path` in a temporary debug snippet.
- Verify Shopify Admin settings: Markets, Languages, Domains, URL handles, redirects.
- Run Shopify CLI theme preview and `shopify theme check`.
- Check Lighthouse only after links resolve, because redirect loops and 404s distort performance metrics.

Solutions:

- Use Shopify's localization form for language/country changes.
- Generate internal links with Shopify objects/routes instead of manually concatenating locale prefixes.
- Add missing translated handles or redirects for renamed resources.
- Keep market-specific links in section settings only when the target exists in that market.
- For menus, use Shopify navigation links and re-save menus after locale/market changes.
- Add monitoring for 404 rates by locale and market in analytics.
