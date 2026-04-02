# API-Referenz

> Automatisch generiert mit `flask generate-api-docs`

**Anzahl Endpoints:** 368

---

## Inhaltsverzeichnis

- [Api Admin](#api-admin) (156 Endpoints)
- [Api Auth](#api-auth) (20 Endpoints)
- [Api Customer](#api-customer) (53 Endpoints)
- [Api Forum](#api-forum) (8 Endpoints)
- [Api Live Location](#api-live-location) (4 Endpoints)
- [Api Messaging](#api-messaging) (38 Endpoints)
- [Api Mobile](#api-mobile) (16 Endpoints)
- [Api Passkey](#api-passkey) (7 Endpoints)
- [Api Platform Admin](#api-platform-admin) (43 Endpoints)
- [Api Public](#api-public) (3 Endpoints)
- [Api Trainer](#api-trainer) (13 Endpoints)
- [Api Twofactor](#api-twofactor) (6 Endpoints)
- [App](#app) (1 Endpoints)

---

## Api Admin

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `GET` | `/api/admin/accounting/jobs` | Sync-Jobs der Schule auflisten (paginiert, filterbar). |
| `POST` | `/api/admin/accounting/jobs/<int:job_id>/retry` | Einzelnen fehlgeschlagenen Job zur Wiederholung vormerken. |
| `POST` | `/api/admin/accounting/retry-all` | Alle fehlgeschlagenen Jobs zur Wiederholung vormerken. |
| `GET` | `/api/admin/accounting/settings` | Buchhaltungs-Konfiguration der Schule lesen. |
| `PUT` | `/api/admin/accounting/settings` | Buchhaltungs-Konfiguration der Schule aktualisieren. |
| `POST` | `/api/admin/accounting/sync-now` | Offene Sync-Jobs der Schule sofort verarbeiten. |
| `POST` | `/api/admin/accounting/test-connection` | Buchhaltungs-Verbindung testen. |
| `GET` | `/api/admin/addons` | Verfuegbare Addons mit Aktivierungsstatus fuer die Schule. |
| `POST` | `/api/admin/addons/<int:addon_id>/toggle` | Addon per ID aktivieren oder deaktivieren (Toggle). |
| `POST` | `/api/admin/addons/<slug>/checkout` | Addon per Stripe aktivieren (recurring) oder Checkout starten (onetime). |
| `POST` | `/api/admin/addons/<slug>/deactivate` | Addon kuendigen (Stripe SubscriptionItem loeschen oder DB deaktivieren). |
| `GET` | `/api/admin/availability` | Alle Verfuegbarkeitsslots der Schule (filterbar). |
| `POST` | `/api/admin/availability` | Verfuegbarkeitsslot fuer einen Trainer erstellen. |
| `PUT` | `/api/admin/availability/<int:slot_id>` | Verfuegbarkeitsslot aktualisieren. |
| `DELETE` | `/api/admin/availability/<int:slot_id>` | Verfuegbarkeitsslot loeschen (nur wenn nicht gebucht). |
| `GET` | `/api/admin/booking-requests` | Alle Buchungsanfragen der Schule auflisten. |
| `PUT` | `/api/admin/booking-requests/<int:req_id>` | Status einer Buchungsanfrage aendern. |
| `POST` | `/api/admin/booking-requests/batch-seen` | Alle neuen Buchungsanfragen als 'gesehen' markieren. |
| `POST` | `/api/admin/bookings/direct` | Erstellt eine Direkt-Buchung: Session + Booking in einem Schritt. |
| `GET` | `/api/admin/cancellation-requests` | Alle Stornierungsanfragen der Schule auflisten. |
| `POST` | `/api/admin/cancellation-requests/<int:req_id>/approve` | Stornoanfrage genehmigen. |
| `POST` | `/api/admin/cancellation-requests/<int:req_id>/reject` | Stornoanfrage ablehnen. |
| `GET` | `/api/admin/course-categories` | Alle Kurs-Kategorien der Schule auflisten. |
| `POST` | `/api/admin/course-categories` | Neue Kurs-Kategorie erstellen. |
| `PUT` | `/api/admin/course-categories/<int:cat_id>` | Kurs-Kategorie aktualisieren. |
| `DELETE` | `/api/admin/course-categories/<int:cat_id>` | Kurs-Kategorie loeschen (nur wenn nicht in Verwendung). |
| `GET` | `/api/admin/course-formats` | Alle Kurs-Formate der Schule auflisten. |
| `POST` | `/api/admin/course-formats` | Neues Kurs-Format erstellen. |
| `PUT` | `/api/admin/course-formats/<int:fmt_id>` | Kurs-Format aktualisieren. |
| `DELETE` | `/api/admin/course-formats/<int:fmt_id>` | Kurs-Format loeschen (nur wenn nicht in Verwendung). |
| `GET` | `/api/admin/courses` | Kursliste der Schule. |
| `POST` | `/api/admin/courses` | Neuen Kurs anlegen. |
| `GET` | `/api/admin/courses/<int:course_id>` | Kurs-Detail. |
| `PUT` | `/api/admin/courses/<int:course_id>` | Kurs bearbeiten. |
| `DELETE` | `/api/admin/courses/<int:course_id>` | Kurs loeschen. |
| `GET` | `/api/admin/courses/<int:course_id>/defaults` | Kurs-Defaults fuer Auto-Fill bei Session-Erstellung. |
| `POST` | `/api/admin/courses/<int:course_id>/duplicate` | Kurs duplizieren — alle Felder kopieren mit '(Kopie)' Suffix. |
| `GET` | `/api/admin/credits` | Alle Kurskarten der Schule auflisten. |
| `POST` | `/api/admin/credits` | Neue Kurskarte fuer einen Hund erstellen. |
| `GET` | `/api/admin/credits/<int:credit_id>` | Kurskarten-Detail abrufen. |
| `PUT` | `/api/admin/credits/<int:credit_id>` | Kurskarte bearbeiten. |
| `POST` | `/api/admin/credits/<int:credit_id>/extend` | Gueltigkeit einer Kurskarte verlaengern. |
| `GET` | `/api/admin/customers/<int:customer_id>/dogs` | Hunde eines Kunden auflisten (fuer Buchungs-UI). |
| `GET` | `/api/admin/dashboard` | Dashboard-Statistiken fuer die Admin-Uebersicht. |
| `GET` | `/api/admin/discount-codes` | Alle Rabattcodes der Schule auflisten. |
| `POST` | `/api/admin/discount-codes` | Neuen Rabattcode erstellen. |
| `GET` | `/api/admin/discount-codes/<int:dc_id>` | Rabattcode-Detail abrufen. |
| `PUT` | `/api/admin/discount-codes/<int:dc_id>` | Rabattcode bearbeiten. |
| `POST` | `/api/admin/discount-codes/<int:dc_id>/toggle` | Rabattcode aktivieren/deaktivieren. |
| `GET` | `/api/admin/dogs` | Alle Hunde der Schule auflisten. |
| `POST` | `/api/admin/dogs` | Neuen Hund anlegen. |
| `GET` | `/api/admin/dogs/<int:dog_id>` | Hund-Detail abrufen. |
| `PUT` | `/api/admin/dogs/<int:dog_id>` | Hund bearbeiten. |
| `DELETE` | `/api/admin/dogs/<int:dog_id>` | Hund loeschen (mit Kaskade). |
| `POST` | `/api/admin/dogs/<int:dog_id>/comments` | Kommentar zu einem Hund hinzufuegen. |
| `DELETE` | `/api/admin/dogs/<int:dog_id>/comments/<int:comment_id>` | Kommentar eines Hundes loeschen. |
| `POST` | `/api/admin/dogs/<int:dog_id>/photo` | Profilbild fuer einen Hund hochladen (multipart/form-data). |
| `GET` | `/api/admin/einzelstunde-requests` | Alle Einzelstunden-Anfragen der Schule auflisten. |
| `POST` | `/api/admin/einzelstunde-requests/<int:req_id>/confirm` | Einzelstunden-Anfrage bestaetigen: Termin + Buchung erstellen. |
| `POST` | `/api/admin/einzelstunde-requests/<int:req_id>/reject` | Einzelstunden-Anfrage ablehnen. |
| `GET` | `/api/admin/email-templates` | Alle E-Mail-Vorlagen auflisten (Defaults + DB-Ueberschreibungen). |
| `GET` | `/api/admin/email-templates/<event_type>` | Eine E-Mail-Vorlage fuer einen Ereignistyp abrufen. |
| `PUT` | `/api/admin/email-templates/<event_type>` | E-Mail-Vorlage speichern oder aktualisieren. |
| `POST` | `/api/admin/email-templates/<event_type>/reset` | E-Mail-Vorlage auf Standard zuruecksetzen (DB-Eintrag loeschen). |
| `GET` | `/api/admin/email-templates/<int:template_id>` | E-Mail-Vorlage per ID abrufen. |
| `PUT` | `/api/admin/email-templates/<int:template_id>` | E-Mail-Vorlage per ID aktualisieren. |
| `DELETE` | `/api/admin/email-templates/<int:template_id>` | E-Mail-Vorlage per ID zuruecksetzen (DB-Eintrag loeschen). |
| `GET` | `/api/admin/invoices` | Rechnungsliste der Schule (paginiert, optional nach Status gefiltert). |
| `POST` | `/api/admin/invoices` | Manuelle Rechnung erstellen. |
| `GET` | `/api/admin/invoices/<int:invoice_id>` | Rechnungsdetail mit Positionen. |
| `GET` | `/api/admin/invoices/<int:invoice_id>/pdf` | Rechnungs-PDF herunterladen. |
| `GET` | `/api/admin/invoices/preview` | Rechnungs-Vorschau als PDF mit Demo-Daten generieren. |
| `POST` | `/api/admin/jobs/daily` | Geburtstags-Mails und Terminerinnerungen (24h) versenden. |
| `GET` | `/api/admin/locations` |  |
| `POST` | `/api/admin/locations` |  |
| `PUT` | `/api/admin/locations/<int:location_id>` |  |
| `DELETE` | `/api/admin/locations/<int:location_id>` |  |
| `GET` | `/api/admin/open-group-approvals` | Alle OpenGroup-Freischaltungen der Schule auflisten. |
| `POST` | `/api/admin/open-group-approvals` | Kunden fuer einen OpenGroup-Kurs freischalten. |
| `POST` | `/api/admin/open-group-approvals/<int:approval_id>/approve` | Bestehende OpenGroup-Freischaltung bestaetigen (Bestaetigungs-Endpunkt). |
| `POST` | `/api/admin/open-group-approvals/<int:approval_id>/reject` | OpenGroup-Freischaltung ablehnen (= widerrufen). |
| `POST` | `/api/admin/open-group-approvals/<int:approval_id>/revoke` | Freischaltung widerrufen. |
| `GET` | `/api/admin/payment-methods` | Zahlungsmethoden des Stripe-Kunden auflisten. |
| `POST` | `/api/admin/payment-methods` | Stripe SetupIntent erstellen fuer neue Zahlungsmethode. |
| `DELETE` | `/api/admin/payment-methods/<pm_id>` | Zahlungsmethode via Stripe entfernen (detach). |
| `PUT` | `/api/admin/payment-methods/<pm_id>/default` | Zahlungsmethode als Standard setzen. |
| `GET` | `/api/admin/payments` | Zahlungen der Schule auflisten. |
| `GET` | `/api/admin/pending-registrations` | Listet alle Kunden mit approved=False fuer die aktuelle Schule. |
| `POST` | `/api/admin/pending-registrations/<int:user_id>/approve` | Alias fuer PUT /users/{id}/approve — iOS-Kompatibilitaet (POST statt PUT). |
| `POST` | `/api/admin/pending-registrations/<int:user_id>/reject` | Lehnt eine ausstehende Kundenregistrierung ab und entfernt die Rolle. |
| `GET` | `/api/admin/referral` | Empfehlungsprogramm-Status abrufen. |
| `POST` | `/api/admin/referral/generate` | Empfehlungscode generieren. |
| `GET` | `/api/admin/reviews` | Alle Bewertungen der Schule. |
| `POST` | `/api/admin/reviews/<int:review_id>/approve` | Review freischalten. |
| `POST` | `/api/admin/reviews/<int:review_id>/reject` | Review ausblenden. |
| `GET` | `/api/admin/sessions` | Sessions der Schule (mit optionalem Datumsfilter). |
| `POST` | `/api/admin/sessions` | Neue Session anlegen. |
| `GET` | `/api/admin/sessions/<int:session_id>` | Session-Detail mit Buchungen. |
| `PUT` | `/api/admin/sessions/<int:session_id>` | Session aktualisieren (Felder, Benachrichtigung bei Aenderung). |
| `DELETE` | `/api/admin/sessions/<int:session_id>` | Session hart loeschen (inkl. Buchungen). |
| `POST` | `/api/admin/sessions/<int:session_id>/cancel` | Session absagen: Buchungen stornieren, Credits zurueckbuchen. |
| `POST` | `/api/admin/sessions/<int:session_id>/makeup` | Nachholstunde fuer eine abgesagte Session erstellen. |
| `POST` | `/api/admin/sessions/<int:session_id>/reschedule` | Abgesagte Session verschieben: neuen Termin setzen, Status reaktivieren. |
| `GET` | `/api/admin/settings` | Schuleinstellungen abrufen. |
| `PUT` | `/api/admin/settings` | Schuleinstellungen aendern. |
| `POST` | `/api/admin/settings/hero-image` | Hero-Image hochladen und optional croppen (Multipart-Upload). |
| `DELETE` | `/api/admin/settings/hero-image` | Hero-Image loeschen. |
| `GET` | `/api/admin/settings/join-code-qr` | QR-Code fuer den Join-Code als base64-PNG zurueckgeben. |
| `POST` | `/api/admin/settings/regenerate-join-code` | Generiert einen neuen Join-Code fuer die Schule. |
| `GET` | `/api/admin/settings/sender-email` | Status der Absender-E-Mail abrufen. |
| `POST` | `/api/admin/settings/sender-email` | Bestaetigungsmail an die gewuenschte Absender-Adresse senden. |
| `DELETE` | `/api/admin/settings/sender-email` | Verifizierte Absender-Adresse entfernen. |
| `POST` | `/api/admin/setup/complete` | Setup-Daten absenden und Setup als abgeschlossen markieren. |
| `POST` | `/api/admin/setup/skip` | Setup ueberspringen — markiert Setup als abgeschlossen. |
| `GET` | `/api/admin/setup/status` | Setup-Status und aktuelle Schuldaten. |
| `GET` | `/api/admin/stripe-connect/dashboard-url` | Stripe Connect Dashboard Login-URL zurueckgeben. |
| `POST` | `/api/admin/stripe-connect/onboard` | Stripe Connect Onboarding starten — gibt URL zurueck statt Redirect. |
| `POST` | `/api/admin/stripe-connect/refresh-status` | Stripe Connect Status von Stripe aktualisieren. |
| `GET` | `/api/admin/stripe-connect/status` | Stripe Connect Account Status der Schule. |
| `POST` | `/api/admin/stripe-portal` | Stripe Customer Portal Session erstellen. |
| `GET` | `/api/admin/subscription` | Aktuellen Abo-Status und Plan-Details zurueckgeben. |
| `POST` | `/api/admin/subscription/cancel` |  |
| `POST` | `/api/admin/subscription/cancel/undo` |  |
| `PUT` | `/api/admin/subscription/plan` | Abo-Plan aendern (starter, professional, enterprise). |
| `GET` | `/api/admin/support-access` | Alle Support-Zugriffs-Anfragen der Schule auflisten. |
| `POST` | `/api/admin/support-access/<int:req_id>/approve` | Support-Zugriff genehmigen (24h Ablauf). |
| `POST` | `/api/admin/support-access/<int:req_id>/reject` | Support-Zugriff ablehnen. |
| `POST` | `/api/admin/support-access/<int:req_id>/revoke` | Genehmigten Support-Zugriff widerrufen. |
| `GET` | `/api/admin/survey-dashboard` | Aggregiertes Survey-Dashboard als JSON. |
| `GET` | `/api/admin/survey-templates` | Alle Umfrage-Templates der Schule. |
| `POST` | `/api/admin/survey-templates` | Neues Umfrage-Template erstellen. |
| `GET` | `/api/admin/survey-templates/<int:template_id>` | Template-Detail mit Fragen. |
| `PUT` | `/api/admin/survey-templates/<int:template_id>` | Template aktualisieren. |
| `DELETE` | `/api/admin/survey-templates/<int:template_id>` | Template loeschen. |
| `GET` | `/api/admin/surveys` | Alle Umfragen der Schule mit Statistiken. |
| `GET` | `/api/admin/surveys/<int:survey_id>` | Umfrage-Detail mit aggregierten Ergebnissen. |
| `GET` | `/api/admin/users` | Nutzerliste der Schule. |
| `POST` | `/api/admin/users` | Neuen Nutzer anlegen und optional Einladung versenden. |
| `GET` | `/api/admin/users/<int:user_id>` | Nutzer-Detail. |
| `PUT` | `/api/admin/users/<int:user_id>` | Nutzer bearbeiten (Name, Rolle, Telefon). |
| `DELETE` | `/api/admin/users/<int:user_id>` | Nutzer loeschen (mit Kaskade). |
| `PUT` | `/api/admin/users/<int:user_id>/approve` | Kundenkonto freischalten (approved=True setzen, E-Mail senden). |
| `POST` | `/api/admin/users/<int:user_id>/link-dog` | Hund mit Nutzer verknuepfen. |
| `POST` | `/api/admin/users/<int:user_id>/resend-invite` | Einladungs-E-Mail erneut senden. |
| `POST` | `/api/admin/users/<int:user_id>/reset-2fa` | 2FA fuer einen Nutzer zuruecksetzen. |
| `POST` | `/api/admin/users/<int:user_id>/send-password-reset` | Passwort-Reset-Mail an Benutzer senden. |
| `POST` | `/api/admin/users/<int:user_id>/unlink-dog/<int:dog_id>` | Hund von Nutzer trennen. |
| `POST` | `/api/admin/video/token` | LiveKit-Token fuer Admin/Trainer generieren. |
| `GET` | `/api/admin/voucher-statistics` | Gutschein-Statistiken der Schule. |
| `GET` | `/api/admin/vouchers` | Alle Gutscheine der Schule auflisten. |
| `POST` | `/api/admin/vouchers` | Neuen Gutschein erstellen. |
| `GET` | `/api/admin/vouchers/<int:voucher_id>` | Gutschein-Detail abrufen. |
| `PUT` | `/api/admin/vouchers/<int:voucher_id>` | Gutschein bearbeiten. |
| `DELETE` | `/api/admin/vouchers/<int:voucher_id>` | Gutschein loeschen. |
| `GET` | `/api/admin/vouchers/<int:voucher_id>/pdf` | Gutschein-PDF herunterladen. |
| `POST` | `/api/admin/vouchers/<int:voucher_id>/send` | Gutschein per E-Mail versenden (mit PDF-Anhang). |

## Api Auth

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `POST` | `/api/auth/2fa/verify` | 2FA-Verifikation nach Login → Token-Paar. |
| `POST` | `/api/auth/avatar` | Avatar hochladen (multipart/form-data). |
| `DELETE` | `/api/auth/avatar` | Avatar löschen. |
| `GET` | `/api/auth/capabilities` | Kanonischer Capability-Contract fuer den aktuellen User-/Schulkontext. |
| `GET` | `/api/auth/confirm/<token>` | E-Mail-Adresse bestätigen. |
| `POST` | `/api/auth/forgot-password` | Passwort-Reset anfordern. |
| `POST` | `/api/auth/login` | JWT-Login: E-Mail + Passwort → Token-Paar. |
| `POST` | `/api/auth/logout` | Logout: Refresh Token widerrufen. |
| `GET` | `/api/auth/me` | Gibt aktuelle User-Daten zurück (mit available_roles). |
| `GET` | `/api/auth/notification-preferences` | Benachrichtigungseinstellungen abrufen. |
| `PUT` | `/api/auth/notification-preferences` | Benachrichtigungseinstellungen aktualisieren. |
| `GET` | `/api/auth/profile/visibility` | Profil-Sichtbarkeitseinstellungen abrufen. |
| `PATCH` | `/api/auth/profile/visibility` | Profil-Sichtbarkeitseinstellungen aktualisieren. |
| `POST` | `/api/auth/refresh` | Refresh Token → neues Access Token. |
| `POST` | `/api/auth/register` | Registrierung: Schule + Admin-User erstellen. |
| `POST` | `/api/auth/register-customer` | Kundenregistrierung per Join-Code (oeffentlich, kein JWT). |
| `POST` | `/api/auth/resend-confirmation` | Bestätigungsemail erneut senden. |
| `POST` | `/api/auth/reset-password` | Passwort mit Token zurücksetzen. |
| `POST` | `/api/auth/set-password` | Passwort setzen (Einladungs-Flow). |
| `PUT` | `/api/auth/switch-context` | Wechselt den aktiven Schul-/Rollen-Kontext (JWT-basiert). |

## Api Customer

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `DELETE` | `/api/customer/account` | Konto loeschen (vollstaendiger Cascade-Delete wie Web-Version). |
| `POST` | `/api/customer/accounting/retry-all-failed` | Alle fehlgeschlagenen Sync-Jobs erneut verarbeiten. |
| `GET` | `/api/customer/accounting/status` | Buchhaltungs-Konfigurationsstatus der Schule. |
| `GET` | `/api/customer/accounting/sync-jobs` | Liste der Sync-Jobs der Schule. |
| `POST` | `/api/customer/accounting/sync-jobs/<int:job_id>/retry` | Einzelnen fehlgeschlagenen Sync-Job erneut verarbeiten. |
| `POST` | `/api/customer/accounting/sync-now` | Sofortige Verarbeitung aller offenen Sync-Jobs. |
| `POST` | `/api/customer/accounting/test-connection` | Verbindungstest zum konfigurierten Buchhaltungstool. |
| `GET` | `/api/customer/available-courses` | Alle Kurse der Schule mit naechstem Termin. |
| `POST` | `/api/customer/blockcourse/<int:course_id>/enroll` | Erstellt ein Blockkurs-Enrollment (Einmalzahlung oder Raten). |
| `POST` | `/api/customer/bookings/<int:booking_id>/cancel` | Buchung stornieren (wenn cancellation_mode == 'free'). |
| `POST` | `/api/customer/bookings/<int:booking_id>/cancel-request` | Stornierungsanfrage stellen. |
| `GET` | `/api/customer/calendar-token` | Kalender-Token und URLs abrufen. |
| `POST` | `/api/customer/change-password` | Passwort aendern. |
| `POST` | `/api/customer/checkout` | Stripe PaymentIntent erstellen fuer iOS SDK. |
| `GET` | `/api/customer/checkout/status/<checkout_id>` | Stripe PaymentIntent-Status abfragen. |
| `GET` | `/api/customer/courses/<int:course_id>` | Kurs-Detail mit Terminen und Freischaltungsstatus. |
| `POST` | `/api/customer/courses/<int:course_id>/request-approval` | Freischaltung fuer einen Opengroup-Kurs beantragen. |
| `GET` | `/api/customer/credits` | Alle aktiven Kurskarten des Kunden. |
| `GET` | `/api/customer/dashboard` | Kunden-Dashboard: Hunde, Buchungen, Guthaben, Kurse. |
| `GET` | `/api/customer/data-export` | DSGVO-konformer Datenexport (Art. 20). |
| `GET` | `/api/customer/dogs` | Alle Hunde des eingeloggten Kunden auflisten. |
| `POST` | `/api/customer/dogs` | Neuen Hund erstellen und dem Kunden zuordnen. |
| `PUT` | `/api/customer/dogs/<int:dog_id>` | Eigenen Hund bearbeiten. |
| `DELETE` | `/api/customer/dogs/<int:dog_id>` | Eigenen Hund loeschen. |
| `POST` | `/api/customer/dogs/<int:dog_id>/files` | Datei fuer eigenen Hund hochladen. |
| `POST` | `/api/customer/dogs/<int:dog_id>/notes-access` | Notiz-Zugriff fuer eine Schule (de-)aktivieren. |
| `POST` | `/api/customer/dogs/<int:dog_id>/photo` | Foto fuer eigenen Hund hochladen. |
| `POST` | `/api/customer/einzelstunde/<int:course_id>/buy-card` | Kauft eine 2er- oder 5er-Karte fuer Einzelstunden. |
| `POST` | `/api/customer/einzelstunde/<int:course_id>/request` | Terminanfrage fuer Einzeltraining stellen. |
| `GET` | `/api/customer/einzelstunde/<int:course_id>/slots` | Verfuegbare Zeitfenster fuer Einzeltraining. |
| `POST` | `/api/customer/einzelstunde/<int:session_id>/book` | Einzelstunde direkt buchen. |
| `POST` | `/api/customer/einzelstunde/anfrage/<int:req_id>/cancel` | Einzelstunden-Anfrage stornieren (nur pending). |
| `POST` | `/api/customer/flexible/<int:course_id>/enroll` | Erstellt ein Flex-Enrollment mit Credits und Zeitfenster. |
| `GET` | `/api/customer/household` | Haushalt mit Mitgliedern, Hunden und offenen Einladungen. |
| `PUT` | `/api/customer/household/billing` | Rechnungsadresse des Haushalts aktualisieren. |
| `POST` | `/api/customer/household/invite` | Mitglied per E-Mail einladen. |
| `POST` | `/api/customer/household/invite/<token>/accept` | Einladung annehmen. |
| `POST` | `/api/customer/household/leave` | Haushalt verlassen. Eigene Hunde werden mitgenommen. |
| `GET` | `/api/customer/invoices` | Liste aller Rechnungen des aktuellen Kunden. |
| `GET` | `/api/customer/invoices/<int:invoice_id>` | Rechnungsdetail mit Positionen. |
| `GET` | `/api/customer/invoices/<int:invoice_id>/pdf` | Rechnungs-PDF herunterladen. |
| `GET` | `/api/customer/my-bookings` | Alle Buchungen und Einzelstunden-Anfragen des Kunden. |
| `GET` | `/api/customer/notification-preferences` | Benachrichtigungspraeferenzen des Kunden abrufen. |
| `PUT` | `/api/customer/notification-preferences` | Benachrichtigungspraeferenzen aktualisieren (Upsert). |
| `GET` | `/api/customer/products` | Verfuegbare Produkte/Kurskarten der Hundeschule. |
| `GET` | `/api/customer/profile` | Profildaten des Kunden. |
| `PUT` | `/api/customer/profile` | Profildaten aktualisieren. |
| `GET` | `/api/customer/sessions` | Verfügbare Sessions der eigenen Schule. |
| `POST` | `/api/customer/sessions/<int:session_id>/book` | Session buchen (block, opengroup, onetime). |
| `POST` | `/api/customer/validate-voucher` | Gutschein-Code pruefen. |
| `POST` | `/api/customer/video/token` | LiveKit-Token fuer Kunden generieren. |
| `GET` | `/api/customer/vouchers` | Gutscheine des Kunden. |
| `POST` | `/api/customer/vouchers/claim` | Gutschein per Code + PIN dem eigenen Konto zuordnen. |

## Api Forum

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `GET` | `/api/forum/categories` | Forum-Kategorien der Schule. |
| `GET` | `/api/forum/categories/<int:category_id>/threads` | Threads in einer Kategorie. |
| `POST` | `/api/forum/categories/<int:category_id>/threads` | Neuen Thread erstellen. |
| `POST` | `/api/forum/post/<int:post_id>/delete` | Post soft-loeschen (Autor oder Admin/Trainer). |
| `GET` | `/api/forum/threads/<int:thread_id>` | Thread mit allen Posts. |
| `POST` | `/api/forum/threads/<int:thread_id>/lock` | Thread sperren/entsperren (nur Admin/Trainer). |
| `POST` | `/api/forum/threads/<int:thread_id>/pin` | Thread pinnen/entpinnen (nur Admin/Trainer). |
| `POST` | `/api/forum/threads/<int:thread_id>/reply` | Antwort auf einen Thread posten. |

## Api Live Location

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `POST` | `/api/messages/<int:conv_id>/live-location` | Live-Location-Session starten (nur Trainer/Admin). |
| `POST` | `/api/messages/<int:conv_id>/toggle-location-sharing` | Location-Sharing fuer Kunden ein-/ausschalten (nur Trainer/Admin). |
| `DELETE` | `/api/messages/live-location/<int:session_id>` | Live-Location-Session stoppen (nur Starter oder Admin). |
| `GET` | `/api/messages/live-location/<int:session_id>/positions` | Aktuelle Positionen einer Live-Location-Session abrufen (jedes Mitglied). |

## Api Messaging

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `GET` | `/api/messages` | Alle Konversationen des Users. |
| `POST` | `/api/messages/<int:conv_id>/archive` | Konversation fuer diesen User archivieren. |
| `DELETE` | `/api/messages/<int:conv_id>/clear` | Alle Nachrichten einer Konversation fuer den aktuellen User loeschen. |
| `POST` | `/api/messages/<int:conv_id>/close` | Konversation schliessen (nur Admin/Trainer). |
| `POST` | `/api/messages/<int:conv_id>/delivered` | Alle Nachrichten einer Konversation als zugestellt markieren. |
| `POST` | `/api/messages/<int:conv_id>/location` | Standort an Konversation senden. |
| `GET` | `/api/messages/<int:conv_id>/media` | Geteilte Medien, Dokumente und Links einer Konversation. |
| `POST` | `/api/messages/<int:conv_id>/mute` | Konversation stummschalten. |
| `DELETE` | `/api/messages/<int:conv_id>/mute` | Stummschaltung aufheben. |
| `POST` | `/api/messages/<int:conv_id>/poll` | Umfrage in Konversation erstellen. |
| `POST` | `/api/messages/<int:conv_id>/read` | Konversation als gelesen markieren. |
| `POST` | `/api/messages/<int:conv_id>/reopen` | Geschlossene Konversation wieder oeffnen (nur Admin/Trainer). |
| `GET` | `/api/messages/<int:conv_id>/search` | Nachrichten in einer Konversation durchsuchen. |
| `POST` | `/api/messages/<int:conv_id>/unarchive` | Konversation fuer diesen User aus dem Archiv holen. |
| `POST` | `/api/messages/<int:conv_id>/upload` | Datei an Konversation senden. |
| `GET` | `/api/messages/<int:conversation_id>` | Nachrichten einer Konversation lesen + als gelesen markieren. |
| `POST` | `/api/messages/<int:conversation_id>` | Nachricht in Konversation senden. |
| `GET` | `/api/messages/<int:conversation_id>/members` | Mitglieder einer Konversation auflisten. |
| `POST` | `/api/messages/<int:conversation_id>/members` | Mitglied zu Konversation hinzufuegen. |
| `DELETE` | `/api/messages/<int:conversation_id>/members/<int:user_id>` | Mitglied aus Konversation entfernen. |
| `DELETE` | `/api/messages/<int:msg_id>` | Nachricht loeschen. Query: ?for=all (global) oder ?for=me (nur fuer mich). |
| `PUT` | `/api/messages/<int:msg_id>/edit` | Eigene Nachricht bearbeiten. |
| `POST` | `/api/messages/<int:msg_id>/reactions` | Emoji-Reaktion auf eine Nachricht. |
| `DELETE` | `/api/messages/<int:msg_id>/reactions/<emoji>` | Emoji-Reaktion entfernen. |
| `POST` | `/api/messages/block/<int:user_id>` | User blockieren. |
| `DELETE` | `/api/messages/block/<int:user_id>` | Block aufheben. |
| `POST` | `/api/messages/broadcast` | Broadcast-Nachricht ueber API erstellen. |
| `GET` | `/api/messages/contacts` | Kontaktliste: alle User der gleichen Schule + Platform-Admin-Erweiterungen. |
| `POST` | `/api/messages/group` | Neue Gruppenkonversation erstellen. |
| `GET` | `/api/messages/info/<int:message_id>` | Zustelldetails einer eigenen Nachricht. |
| `GET` | `/api/messages/muted` | Liste aller stummgeschalteten Konversationen. |
| `POST` | `/api/messages/new` | Neue Direktnachricht starten. |
| `POST` | `/api/messages/polls/<int:poll_id>/close` | Umfrage manuell schliessen. |
| `POST` | `/api/messages/polls/<int:poll_id>/respond` | Antwort auf Umfrage abgeben. |
| `GET` | `/api/messages/polls/<int:poll_id>/results` | Umfrage-Ergebnisse abrufen. |
| `POST` | `/api/messages/report/<int:user_id>` | User melden. |
| `GET` | `/api/messages/shared-courses/<int:user_id>` | Gemeinsame Kurse zwischen aktuellem User und Ziel-User. |
| `GET` | `/api/messages/unread-count` | Gesamte ungelesene Nachrichten des Users. |

## Api Mobile

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `GET` | `/api/customer/bookings` | Alle Buchungen des Kunden. |
| `GET` | `/api/customer/dashboard` | Dashboard-Daten des Kunden: Hunde, nächste Termine, Buchungen. |
| `GET` | `/api/dogs/<int:dog_id>/comments` | Kommentare eines Hundes abrufen (nur Schul-Kommentare). |
| `POST` | `/api/dogs/<int:dog_id>/comments` | Neuen Kommentar zu einem Hund anlegen (nur Admin/Trainer). |
| `DELETE` | `/api/dogs/<int:dog_id>/comments/<int:comment_id>` | Eigenen Kommentar loeschen. |
| `GET` | `/api/dogs/<int:dog_id>/files` | Dateien eines Hundes abrufen. |
| `DELETE` | `/api/dogs/<int:dog_id>/files/<int:file_id>` | Eigene Datei loeschen. |
| `POST` | `/api/dogs/<int:dog_id>/files/<int:file_id>/toggle-share` | Freigabe-Status einer Datei umschalten. |
| `POST` | `/api/dogs/<int:dog_id>/files/upload` | Datei fuer einen Hund hochladen (multipart/form-data). |
| `POST` | `/api/push/register` | Push-Token registrieren (APNs/FCM). |
| `POST` | `/api/push/unregister` | Push-Token entfernen (z.B. bei Logout aus App). |
| `GET` | `/api/session-location/<int:course_id>` | Standort der naechsten Session eines Kurses. |
| `POST` | `/api/trainer/attendance/<int:booking_id>` | Anwesenheit per JSON markieren (für Offline-Queue-Sync). |
| `GET` | `/api/trainer/session/<int:session_id>` | Session-Detail mit Buchungsliste. |
| `GET` | `/api/trainer/today` | Heutige Sessions des Trainers mit Teilnehmern. |
| `GET` | `/api/trainer/week` | Wochenübersicht des Trainers. |

## Api Passkey

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `DELETE` | `/api/auth/passkey/<int:passkey_id>` | Entfernt einen registrierten Passkey. |
| `PATCH` | `/api/auth/passkey/<int:passkey_id>` | Benennt einen Passkey um. |
| `GET` | `/api/auth/passkey/list` | Gibt alle registrierten Passkeys des Nutzers zurück. |
| `POST` | `/api/auth/passkey/login/begin` | Startet Passkey-Login — gibt Challenge + Options zurück. |
| `POST` | `/api/auth/passkey/login/complete` | Verifiziert Passkey-Response und gibt JWT-Tokens zurück. |
| `POST` | `/api/auth/passkey/register/begin` | Startet Passkey-Registration für eingeloggten User. |
| `POST` | `/api/auth/passkey/register/complete` | Speichert neuen Passkey-Credential. |

## Api Platform Admin

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `GET` | `/api/platform/blog` | Alle Blog-Artikel auflisten. |
| `POST` | `/api/platform/blog` | Neuen Blog-Artikel erstellen. |
| `GET` | `/api/platform/blog/<int:post_id>` | Blog-Artikel abrufen. |
| `PUT` | `/api/platform/blog/<int:post_id>` | Blog-Artikel bearbeiten. |
| `DELETE` | `/api/platform/blog/<int:post_id>` | Blog-Artikel loeschen. |
| `POST` | `/api/platform/blog/<int:post_id>/publish` | Blog-Artikel veroeffentlichen/zurueckziehen. |
| `GET` | `/api/platform/dashboard` | KPIs fuer das Platform-Admin-Dashboard. |
| `GET` | `/api/platform/email-templates` | Alle plattformweiten E-Mail-Vorlagen auflisten. |
| `PUT` | `/api/platform/email-templates/<event_type>` | Plattformweite E-Mail-Vorlage aktualisieren. |
| `GET` | `/api/platform/finances` | Finanz-Uebersicht: MRR, Plan-Verteilung, Umsatz-Statistiken. |
| `GET` | `/api/platform/invoices` | Paginierte Liste aller Plattform-Rechnungen. |
| `GET` | `/api/platform/invoices/<int:invoice_id>/pdf` | Rechnungs-PDF herunterladen. |
| `POST` | `/api/platform/leave-school` | Schul-Admin-Bereich verlassen. |
| `GET` | `/api/platform/logs` | Audit-Logs abrufen (paginiert, Typ-Filter). |
| `GET` | `/api/platform/maintenance` | Aktuellen Wartungsmodus-Status abrufen. |
| `PUT` | `/api/platform/maintenance` | Wartungsmodus setzen oder aktualisieren. |
| `POST` | `/api/platform/maintenance/schedule` | Wartungsfenster planen. |
| `DELETE` | `/api/platform/maintenance/schedule` | Geplantes Wartungsfenster loeschen. |
| `GET` | `/api/platform/prices` | Alle Plan- und Addon-Preise auflisten. |
| `PUT` | `/api/platform/prices/addons/<int:price_id>` | Addon-Preis aendern: neuen Stripe Price erstellen, alten deaktivieren. |
| `PUT` | `/api/platform/prices/plans/<plan_id>` | Plan-Preis aendern: neuen Stripe Price erstellen, alten deaktivieren. |
| `PUT` | `/api/platform/prices/plans/<plan_id>/content` | Plan-Texte und Listeninhalte aktualisieren. |
| `GET` | `/api/platform/promo-codes` | Alle Promo-Codes auflisten. |
| `POST` | `/api/platform/promo-codes` | Neuen Promo-Code erstellen. |
| `GET` | `/api/platform/promo-codes/<int:code_id>` | Promo-Code-Detail mit Einloesungen abrufen. |
| `PUT` | `/api/platform/promo-codes/<int:code_id>` | Promo-Code bearbeiten. |
| `POST` | `/api/platform/promo-codes/<int:code_id>/toggle` | Promo-Code aktivieren/deaktivieren. |
| `GET` | `/api/platform/schools` | Paginierte Schulliste mit Filtern (?q=, ?plan=, ?status=, ?page=, ?per_page=). |
| `POST` | `/api/platform/schools` | Schule manuell anlegen (optional mit Admin-User). |
| `GET` | `/api/platform/schools/<int:school_id>` | Schul-Detailansicht. |
| `PUT` | `/api/platform/schools/<int:school_id>` | Schulfelder aktualisieren. |
| `DELETE` | `/api/platform/schools/<int:school_id>` | Schule loeschen (mit Cascade). |
| `POST` | `/api/platform/schools/<int:school_id>/enter` | Schul-Admin-Bereich betreten (nur nach Genehmigung). |
| `POST` | `/api/platform/schools/<int:school_id>/request-access` | Support-Zugriffsanfrage fuer eine Hundeschule stellen. |
| `POST` | `/api/platform/schools/<int:school_id>/set-plan` | Abo-Plan einer Schule aendern. |
| `GET` | `/api/platform/settings` | Plattform-Einstellungen abrufen. |
| `PUT` | `/api/platform/settings` | Plattform-Einstellungen aktualisieren. |
| `POST` | `/api/platform/settings/test-graph` | Microsoft Graph API Verbindung testen (Token-Request). |
| `GET` | `/api/platform/system` | System-Health-Informationen (Disk, DB-Groesse, Python-Version). |
| `GET` | `/api/platform/users` | Alle Plattform-Administratoren auflisten. |
| `POST` | `/api/platform/users` | Neuen Plattform-Administrator erstellen. |
| `POST` | `/api/platform/users/<int:user_id>/reset-2fa` | 2FA fuer einen Admin zuruecksetzen. |
| `POST` | `/api/platform/users/<int:user_id>/revoke` | Plattform-Admin-Zugang entziehen. |

## Api Public

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `GET` | `/api/public/pricing` | Kanonischen Pricing-Katalog oeffentlich ausliefern. |
| `GET` | `/api/public/schools/by-code/<code>` | Schul-Lookup per Join-Code. Oeffentlich, kein JWT noetig. |
| `GET` | `/api/public/schools/search` | Schulsuche per Name (ilike). Oeffentlich, kein JWT noetig. |

## Api Trainer

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `POST` | `/api/trainer/attendance/<int:booking_id>` | Anwesenheit fuer eine Buchung markieren. |
| `GET` | `/api/trainer/availability` | Eigene Verfuegbarkeits-Slots abrufen. |
| `POST` | `/api/trainer/availability` | Neuen Verfuegbarkeits-Slot erstellen. |
| `PUT` | `/api/trainer/availability/<int:slot_id>` | Verfuegbarkeits-Slot aendern (nur eigene, nicht gebuchte). |
| `DELETE` | `/api/trainer/availability/<int:slot_id>` | Verfuegbarkeits-Slot loeschen (nur eigene, nicht gebuchte). |
| `GET` | `/api/trainer/blockings` | Eigene Blockierungen/Urlaube abrufen. |
| `POST` | `/api/trainer/blockings` | Neue Blockierung/Urlaub erstellen. |
| `DELETE` | `/api/trainer/blockings/<int:blocking_id>` | Eigene Blockierung loeschen. |
| `POST` | `/api/trainer/book-customer` | Trainer bucht einen Kunden direkt ein (Session + Booking). |
| `GET` | `/api/trainer/calendar` | Monatskalender mit Sessions, Verfuegbarkeit und Blockierungen. |
| `GET` | `/api/trainer/customer-dogs` | Hunde eines Kunden fuer die Direktbuchungs-Auswahl. |
| `GET` | `/api/trainer/dashboard` | Trainer-Dashboard: heutige + kommende Sessions. |
| `POST` | `/api/trainer/video/token` | LiveKit-Token fuer Trainer generieren. |

## Api Twofactor

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `POST` | `/api/auth/2fa/backup-codes` | Neue Backup-Codes generieren. |
| `POST` | `/api/auth/2fa/disable-all` | Alle 2FA-Methoden deaktivieren (erfordert Passwort). |
| `GET` | `/api/auth/2fa/status` | Aktueller 2FA-Status. |
| `POST` | `/api/auth/2fa/totp/disable` | TOTP deaktivieren. |
| `POST` | `/api/auth/2fa/totp/setup` | TOTP-Setup: Generiert Secret und QR-URI. |
| `POST` | `/api/auth/2fa/totp/verify-setup` | TOTP-Code verifizieren und 2FA aktivieren. |

## App

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| `GET` | `/api/schools/join/<code>` |  |

---

## Authentifizierung

Alle Endpoints (ausser `/api/auth/login`, `/api/auth/register`) 
erfordern einen JWT-Token im `Authorization`-Header:

```
Authorization: Bearer <access_token>
```

Token werden ueber `POST /api/auth/login` bezogen.
