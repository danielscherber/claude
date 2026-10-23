# JSON-LD Schema-Vorlagen

Alle Werte in `<…>` sind Platzhalter. Keine echten Kunden-, Personen- oder Zugangsdaten in diese Datei schreiben.

Einbau in Webflow: Seiten-Settings → Custom Code → **Inside `<head>` tag**, jeweils in `<script type="application/ld+json"> … </script>`.
Globale Schemas (Organization, LocalBusiness, WebSite) einmalig in den Site-Settings → Custom Code → Head.

Validieren: `https://validator.schema.org/` und `https://search.google.com/test/rich-results`.

---

## Organization + LocalBusiness (global)

```json
{
  "@context": "https://schema.org",
  "@type": ["Organization", "LocalBusiness"],
  "@id": "https://<domain>/#organization",
  "name": "<Firmenname>",
  "url": "https://<domain>/",
  "logo": "https://<domain>/<logo>.png",
  "image": "https://<domain>/<teamfoto>.jpg",
  "description": "<Ein Satz: was, für wen, wo>",
  "telephone": "+49 <Nummer>",
  "email": "<kontakt>@<domain>",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "<Straße Nr.>",
    "postalCode": "<PLZ>",
    "addressLocality": "<Ort>",
    "addressRegion": "<Bundesland>",
    "addressCountry": "DE"
  },
  "geo": { "@type": "GeoCoordinates", "latitude": 0.0, "longitude": 0.0 },
  "areaServed": ["<Ort>", "<Region>"],
  "foundingDate": "<JJJJ>",
  "sameAs": [
    "https://www.linkedin.com/company/<slug>",
    "https://www.youtube.com/@<handle>",
    "https://www.instagram.com/<handle>"
  ]
}
```

NAP (Name, Adresse, Telefon) muss exakt mit Google Business Profile und Verzeichnissen übereinstimmen.

## WebSite (Startseite)

```json
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "@id": "https://<domain>/#website",
  "url": "https://<domain>/",
  "name": "<Firmenname>",
  "inLanguage": "de-DE",
  "publisher": { "@id": "https://<domain>/#organization" }
}
```

## BreadcrumbList (alle Unterseiten)

```json
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    { "@type": "ListItem", "position": 1, "name": "Start", "item": "https://<domain>/" },
    { "@type": "ListItem", "position": 2, "name": "<Bereich>", "item": "https://<domain>/<bereich>" },
    { "@type": "ListItem", "position": 3, "name": "<Seite>" }
  ]
}
```

## Service (Leistungsseiten)

```json
{
  "@context": "https://schema.org",
  "@type": "Service",
  "name": "<Leistung>",
  "serviceType": "<Kategorie, z.B. Videoproduktion>",
  "description": "<BLUF-Satz aus der Seite>",
  "provider": { "@id": "https://<domain>/#organization" },
  "areaServed": { "@type": "City", "name": "<Ort>" },
  "offers": {
    "@type": "Offer",
    "priceSpecification": {
      "@type": "PriceSpecification",
      "minPrice": 0,
      "maxPrice": 0,
      "priceCurrency": "EUR"
    }
  }
}
```

Preisangaben nur, wenn sie auch sichtbar auf der Seite stehen.

## FAQPage (Leistungsseiten mit FAQ)

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "<Frage exakt wie auf der Seite>",
      "acceptedAnswer": { "@type": "Answer", "text": "<Antwort exakt wie auf der Seite>" }
    }
  ]
}
```

Hinweis: Google zeigt FAQ-Rich-Results seit 2023 nur noch für wenige autoritative Seiten. Das Markup bleibt trotzdem sinnvoll, weil es Frage-Antwort-Paare maschinenlesbar macht.

## VideoObject (Seiten mit Video)

```json
{
  "@context": "https://schema.org",
  "@type": "VideoObject",
  "name": "<Videotitel>",
  "description": "<Worum geht es, 1–2 Sätze>",
  "thumbnailUrl": "https://<domain>/<thumbnail>.jpg",
  "uploadDate": "<JJJJ-MM-TT>",
  "duration": "PT<M>M<S>S",
  "contentUrl": "https://<video-url>",
  "embedUrl": "https://www.youtube.com/embed/<id>",
  "transcript": "<Volltranskript oder Link>",
  "publisher": { "@id": "https://<domain>/#organization" }
}
```

## Article + Person (Blog / Ratgeber)

```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "<Titel als Frage>",
  "description": "<BLUF-Satz>",
  "image": "https://<domain>/<bild>.jpg",
  "datePublished": "<JJJJ-MM-TT>",
  "dateModified": "<JJJJ-MM-TT>",
  "inLanguage": "de-DE",
  "author": {
    "@type": "Person",
    "name": "<Vorname Nachname>",
    "jobTitle": "<Funktion>",
    "url": "https://<domain>/team/<slug>",
    "sameAs": ["https://www.linkedin.com/in/<slug>"]
  },
  "publisher": { "@id": "https://<domain>/#organization" }
}
```

`dateModified` bei jeder inhaltlichen Änderung aktualisieren — im Webflow-CMS am besten aus dem Feld „Updated On“ befüllen.
