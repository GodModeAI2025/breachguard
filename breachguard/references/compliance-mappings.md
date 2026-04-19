# Compliance-Regulierungs-Mappings

Die `compliance`-Domaene hat 56 Lenses mit starker regulatorischer Kopplung.
Diese Referenz mappt Lens-IDs auf konkrete Regulierungen, Gesetze und Standards.

**Geltungsbereich:** primaer EU/DE, punktuell US (HIPAA, FERPA, EO 14028).

## Mapping-Tabelle

| Lens-ID | Regulierung / Gesetz / Standard |
|---------|-------------------------------|
| `accessibility-eaa` | EAA 2025 (EU) 2019/882 · WCAG 2.2 AA · BITV 2.0 · BFSG |
| `ai-act` | AI Act (EU) 2024/1689 |
| `algorithmic-discrimination` | AI Act Art. 9-15 · AGG |
| `aml-kyc` | AMLD6 · MLD 2024/1640 · GwG |
| `audit-trail-gobd` | GoBD · AO § 147 |
| `automated-decisions` | GDPR Art. 22 · AI Act Art. 6, 14 |
| `bnpl-credit` | Consumer Credit Directive 2023/2225 · VerbrKrRL |
| `clinical-trial-data` | EU-CTR 536/2014 · GCP |
| `consent-flows` | GDPR Art. 6(1)(a), 7 · ePrivacy |
| `cookie-policy` | ePrivacy Art. 5(3) · GDPR Art. 6 · TTDSG |
| `cyber-resilience-act` | CRA (EU) 2024/2847 |
| `data-retention` | GDPR Art. 5(1)(e), 17 · BDSG § 35 |
| `diga-health-app` | DVG · DiGAV · MDR |
| `digital-content-conformity` | DCD 2019/770/EU |
| `dispute-resolution` | ADR Directive 2013/11/EU |
| `dora-operational-resilience` | DORA (EU) 2022/2554 |
| `ecommerce-law` | ECRL 2000/31/EC · DSA |
| `education-data` | FERPA (US) · Landes-Datenschutzgesetze (DE) |
| `eidas-signatures` | eIDAS 910/2014 · eIDAS 2.0 |
| `employee-monitoring` | GDPR Art. 88 · BDSG § 26 · BetrVG § 87 |
| `food-labeling` | LMIV (EU) 1169/2011 · LMKV |
| `gambling-compliance` | GluStV 2021 (DE) · 5. GluStV |
| `gdpr-dsgvo` | GDPR Art. 5-99 (complete) · DSGVO |
| `geoblocking` | Geoblocking Regulation (EU) 2018/302 |
| `hipaa-health-data` | HIPAA (US) · HITECH |
| `impressum` | TMG § 5 · DDG § 5 |
| `incident-response` | NIS2 Art. 23 · GDPR Art. 33-34 · BSIG § 8b |
| `invoice-compliance` | UStG § 14 · E-Rechnung 2025 (DE) |
| `kritis-infrastructure` | BSI-KritisV · NIS2 Art. 21 · IT-SiG 2.0 |
| `mdr-medical-device` | MDR (EU) 2017/745 · IVDR |
| `newsletter-consent` | GDPR Art. 6(1)(a) · UWG § 7 |
| `nis2` | NIS2 Directive (EU) 2022/2555 Art. 20-23 · NIS2UmsuCG |
| `pay-transparency` | Pay Transparency Directive (EU) 2023/970 · EntgTranspG |
| `payment-pci` | PCI DSS v4.0 |
| `personalized-pricing` | CRD Art. 6 · Omnibus |
| `platform-fairness` | P2B Regulation (EU) 2019/1150 · DMA |
| `price-transparency` | PAngV · Price Indication Directive |
| `privacy-by-design` | GDPR Art. 25 |
| `privacy-policy-audit` | GDPR Art. 13, 14 |
| `product-liability` | Product Liability Directive (EU) 2024/2853 · ProdHaftG |
| `psd2-strong-auth` | PSD2 RTS SCA (EU) 2018/389 |
| `refund-widerrufsrecht` | CRD 2011/83/EU · BGB § 312g |
| `review-authenticity` | Omnibus 2019/2161 · UCPD |
| `sbom-supply-chain` | CRA Annex I · EO 14028 (US) · NIS2 Art. 21(2) |
| `secure-sdlc` | NIST SSDF · BSIMM · ISO 27034 |
| `security-disclosure` | CRA Art. 11 · NIS2 Art. 23 |
| `smart-meter-data` | MsbG · Smart Meter Gateway (BSI-TR-03109) |
| `sovereignty` | EUCS · BSI C5 · Gaia-X |
| `subscription-cancellation` | Omnibus 2019/2161 · BGB § 312k |
| `time-tracking` | ECJ C-55/18 · ArbZG (DE nach ECJ) |
| `tos-legal-audit` | UCPD · Unfair Contract Terms Directive 93/13/EEC |
| `unfair-practices` | UCPD 2005/29/EC · UWG |
| `vehicle-cybersecurity` | UNECE R155 · R156 · ISO/SAE 21434 |
| `whistleblower-protection` | Whistleblower Directive 2019/1937 · HinSchG |
| `youth-protection` | JMStV · JuSchG · DSA Art. 28 |

## Cluster-Sicht

### GDPR/DSGVO-Cluster
Alle Datenverarbeitungs-relevanten Lenses. Bei einem GDPR-Audit alle davon
pruefen: `gdpr-dsgvo`, `consent-flows`, `cookie-policy`, `data-retention`,
`privacy-by-design`, `privacy-policy-audit`, `automated-decisions`,
`employee-monitoring`, `newsletter-consent`.

### Cybersecurity-Cluster (KRITIS/NIS2/CRA/DORA)
Fuer Betreiber kritischer/wichtiger Einrichtungen in DE/EU:
`kritis-infrastructure`, `nis2`, `incident-response`,
`dora-operational-resilience`, `cyber-resilience-act`, `sbom-supply-chain`,
`security-disclosure`.

### Accessibility-Cluster (BFSG/EAA/WCAG)
Gilt ab 28.06.2025 (BFSG) fuer viele kommerzielle Angebote:
`accessibility-eaa` plus Lenses in der `adaptive-ux`-Domaene
(`theme-adaptation`, `rtl-layout`, `print-stylesheet`, `viewport-sizing`).

### Finanzdienstleistungen
`payment-pci`, `psd2-strong-auth`, `aml-kyc`, `bnpl-credit`, `dora-operational-resilience`.

### Health/Medical
`hipaa-health-data`, `diga-health-app`, `clinical-trial-data`,
`mdr-medical-device`.

### E-Commerce/Consumer
`impressum`, `ecommerce-law`, `refund-widerrufsrecht`, `price-transparency`,
`subscription-cancellation`, `geoblocking`, `personalized-pricing`,
`platform-fairness`, `review-authenticity`, `unfair-practices`,
`digital-content-conformity`, `tos-legal-audit`, `dispute-resolution`,
`invoice-compliance`.

### AI/Automatisierte Entscheidungen
`ai-act`, `algorithmic-discrimination`, `automated-decisions`.

### Arbeitsrecht/Sozial
`pay-transparency`, `time-tracking`, `whistleblower-protection`,
`employee-monitoring`, `youth-protection`.

### Sektor-spezifisch
`audit-trail-gobd` (Steuern/Buchhaltung), `eidas-signatures`,
`education-data`, `food-labeling`, `gambling-compliance`,
`product-liability`, `secure-sdlc`, `smart-meter-data`, `sovereignty`,
`vehicle-cybersecurity`.

## Verwendung im Compliance-Audit

**Anwendungsfall:** User fragt "pruefe unsere App auf GDPR-Compliance".

Vorgehen:
1. Mapping-Tabelle oben verwenden, um relevante Lenses zu finden
   (GDPR-Cluster)
2. `references/lenses/compliance.md` laden
3. Jede Lens auf Code/Config anwenden
4. Findings mit `regulation`-Feld im `finding`-Schema taggen (siehe
   `agent-output.md`)
5. `compliance_matrix`-Output produzieren statt Standard-Audit-Report

## Disclaimer

Die Mappings sind **Orientierung**, kein Rechtsrat. Regulierungen aendern
sich; Versionen in der Tabelle sind Stand **2026-04-18**.
Bei konkreten Compliance-Fragen fachliche Beratung hinzuziehen.
