# Venezia.pl × Luigi's Box Recommender

Technical assignment for the **Integrations Consultant / Solutions Engineer** role at [Luigi's Box](https://www.luigisbox.com/).

The brief: design on-site recommenders for [venezia.pl](https://www.venezia.pl/) — model choice, a Recommender API request, and a frontend widget.

**Live widget:** [lenarkuba.github.io/luigisbox-venezia-recommender-case-study](https://lenarkuba.github.io/luigisbox-venezia-recommender-case-study/)

You can also open [`index.html`](index.html) locally. No build step.

Docs used:

- [Reference models](https://docs.luigisbox.com/recommendations/models/)
- [Choosing recommendation models](https://docs.luigisbox.com/quickstart/recommendations/choosing-models/)
- [Recommender API](https://docs.luigisbox.com/recommendations/api/v1/recommender/)
- [Recommender tutorial](https://docs.luigisbox.com/tutorials/recommender/)

---

## Placement → model

| Placement | Goal | `recommendation_type` |
| --- | --- | --- |
| Product detail | Increase AOV | `item_detail_alternatives` + `item_detail_complements` |
| Basket page | Encourage add-ons | `basket` |
| Homepage | Personalized return visit | `user_click_based` |
| Homepage, new visitor | Avoid an empty box | `trends` (fallback) |

`item_detail_alternatives` is the upsell model: similar products, preferring slightly more expensive ones. Complements add cheaper extras (suede care, insoles, a bag). On the basket page the whole cart matters, so the model is `basket`. Returning homepage visitors need browsing history (`user_click_based`); cold start uses `trends`.

---

## Task 1 — client email (Polish)

**Temat:** Rekomendacje na Venezia.pl — propozycja modeli

Cześć,

Poniżej krótka propozycja modeli Luigi’s Box na trzy miejsca na stronie, dopasowana do Waszych celów.

**1. Karta produktu — wzrost średniej wartości zamówienia**  
Modele: `item_detail_alternatives` oraz `item_detail_complements`  
Boks „Może Ci się spodobać” (`item_detail_alternatives`) pokazuje podobne modele obuwia i faworyzuje te trochę droższe. Klient, któremu aktualne botki nie pasują w 100%, zostaje przy zakupie i często wybiera wyższy wariant cenowy. Drugi boks, „Często kupowane razem” (`item_detail_complements`), dokłada tańsze dodatki — impregnat do zamszu, wkładki, torebkę. To ten sam cel (wyższe AOV), tylko inną drogą: upsell vs. doposażenie.

**2. Koszyk — zachęta do dodatków**  
Model: `basket`  
Na stronie koszyka nie pokazujemy kolejnych par butów — to odciąga od kasy. Model `basket` patrzy na cały koszyk i proponuje uzupełnienia do całego zamówienia, zwykle tańsze. To jest właściwe miejsce na add-on, nie na zamiennik.

**3. Strona główna — spersonalizowany powrót**  
Model: `user_click_based`  
Powracający klient od razu widzi produkty zbliżone do tych, które ostatnio oglądał, ale których nie kupił. Skraca to drogę z homepage z powrotem do konkretnego fasonu, zamiast zaczynać przeglądanie od zera.

**Fallback dla nowych użytkowników (brak historii):**  
Dla nowej sesji bez historii kliknięć ten sam slot na homepage przełączamy na `trends` — najpopularniejsze produkty w sklepie. Pusty boks nie wchodzi w grę, a ten model nie wymaga profilu użytkownika.

Dajcie znać, czy idziemy w tę konfigurację. W kolejnym kroku mogę przygotować zapytania API i sposób wpięcia przez GTM — Cookiebot już macie na stronie, więc `user_id` wysyłamy tylko po zgodzie na analitykę / personalizację.

Pozdrawiam  
Jakub Lenartowicz

---

## Task 2 — Recommender API

`POST https://live.luigisbox.com/v1/recommend?tracker_id=179075-204259`

Body is a **JSON array** so two widgets on the same PDP travel in one call (lower latency, one user-profile load, automatic product dedup).

Seed product from the live `view_item` dataLayer event: SKU `V564-03938-02TDM` ([PDP](https://www.venezia.pl/brazowe-zamszowe-botki-damskie-z-elastycznymi-wstawkami-v564-03938-02tdm)). Magento also exposes `product_id` `447705`; that would only be used if the catalog is indexed on the entity id.

```bash
curl -X POST "https://live.luigisbox.com/v1/recommend?tracker_id=179075-204259" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "Accept-Encoding: gzip, deflate" \
  -d '[
    {
      "recommendation_type": "item_detail_alternatives",
      "recommender_client_identifier": "pdp_you_may_also_like",
      "item_ids": ["V564-03938-02TDM"],
      "size": 6,
      "user_id": "LB_USER_ID",
      "hit_fields": ["title", "url", "price", "labels", "image_link"],
      "recommendation_context": {
        "brand": {
          "values": ["Venezia Outlet"],
          "operator": "not"
        }
      }
    },
    {
      "recommendation_type": "last_seen",
      "recommender_client_identifier": "pdp_last_seen",
      "item_ids": [],
      "size": 6,
      "user_id": "LB_USER_ID",
      "hit_fields": ["title", "url", "price", "labels", "image_link"]
    }
  ]'
```

| Parameter | Why it is there |
| --- | --- |
| `tracker_id` | Public site id in the query string. This endpoint is unauthenticated. |
| `recommendation_type` | Model to run. The second widget uses `last_seen` (Recently visited). |
| `recommender_client_identifier` | Widget name for analytics. Distinct from the model name. |
| `item_ids` | Catalog identity of the PDP product. Must match the feed and analytics. |
| `size` | Six cards, as requested. |
| `user_id` | `_lb` cookie after Cookiebot consent, or Magento `customerId` when logged in. Omit without consent. |
| `hit_fields` | Only fields the widget renders. |
| `recommendation_context` | Request-time filter. Brief: exclude brand `"Venezia Outlet"`. |

On the live site the brand is `Venezia` and Outlet is a **category** (`item_category: "OUTLET"`). The seed product itself sits in Outlet. In a kickoff I would confirm the catalog field and, if it matches today's dataLayer, filter category `OUTLET` instead of brand. “Never show Outlet” on an Outlet PDP is a business call, not only a filter.

This `tracker_id` is valid Luigi's Box, but it is not Venezia's catalog. The request is structurally correct; a live 404 on recommender mapping would mean models are not trained on this tracker.

---

## Task 3 — frontend widget

Vanilla HTML / CSS / JS in [`index.html`](index.html). No React — recommendation boxes are usually injected into Magento / GTM with a small script.

- Renders into `#lbx-recommender`
- Each hit is a clickable card: image, labels, title, price
- Relative URLs are prefixed with `https://www.venezia.pl`
- `is_new` → Nowość, `is_sale` → Promocja
- Missing `image_link` uses an SVG placeholder; broken URLs do the same via `onerror`
- Text goes through `textContent` so API strings are not interpreted as HTML
- Empty `hits` hides the container

The Task 3 payload uses made-up paths such as `/brazowe-sztyblety-damskie-v102-11003-03`. Magento on venezia.pl treats an unknown URL as a search query, so a raw click would land on „Wyniki wyszukiwania”. The demo maps those slugs to live product pages. Assignment image URLs (`/img/11001.jpg`) are also fictional, so the placeholder is expected until the API returns a real `image_link`.

Production still needs `view_item_list` and `select_item` (dataLayer or Events API). Without those events the models cannot learn. This snippet only covers rendering, which is what Task 3 asked for.

Venezia already has GTM and Cookiebot. The practical integration path is GTM plus the Luigi's Box collector, with `user_id` sent only when `analytics_storage` / `personalization_storage` is granted.
