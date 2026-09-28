# HTTPS für die privaten Webdienste

Im Film bleiben Hermes, Vikunja und Papra unter `http://<name>.agent.internal:<port>` erreichbar.
Die Dienste lauschen auf der NetBird-Adresse; die Hetzner Firewall hat keine eingehende Regel.
NetBird verschlüsselt den Transport zwischen den Peers. Im Browser ist die Verbindung zur
Anwendung trotzdem HTTP und die Portnummer bleibt sichtbar.

Wenn HTTPS im Browser gebraucht wird, muss ein Reverse Proxy vor die Anwendungen. Er kann
auch die Portnummer aus der Adresse entfernen. Vorher entscheiden, wie Zertifikate ausgestellt
und auf den Geräten als vertrauenswürdig erkannt werden sollen:

1. **Eigene öffentliche Domain, DNS-Challenge:** Einen Namen aus einer eigenen Domain
   verwenden und Caddy per DNS-Challenge ein öffentlich vertrauenswürdiges Zertifikat
   ausstellen lassen. Für die Zertifikatsprüfung sind keine offenen HTTP-Ports nötig.
   Der DNS-Provider muss eine passende API unterstützen; das Caddy-DNS-Modul, Zugangstoken,
   Erneuerung und private Namensauflösung müssen gepflegt werden. Der Reverse Proxy kann
   weiterhin ausschließlich auf der NetBird-Adresse lauschen.
2. **Interne Zertifizierungsstelle:** Caddy kann für private Namen Zertifikate mit einer
   internen CA ausstellen. Deren Root-Zertifikat muss auf jedem verwendeten Endgerät bewusst
   installiert und vertraut werden. Wer dieses Root-Zertifikat verteilt oder verliert, hat
   einen eigenen Betriebs- und Sicherheitsfall.
3. **NetBird Reverse Proxy:** Die Produktfunktion kann Domain und TLS verwalten. Vor der
   Freigabe den konkreten Zugriffsmodus prüfen und *NetBird-Only Access* aktivieren, wenn der
   Dienst nur für Peers erreichbar sein soll. Einen öffentlich erreichbaren Proxy-Link nicht
   mit der privaten Peer-Adresse verwechseln.

Die im Video gezeigte Variante braucht keinen zusätzlichen Reverse Proxy. Sie ist für
dieses persönliche Setup eine bewusste Abwägung; sie ersetzt keine Bewertung der
Anwendungsrechte, Tokens oder Geräte, die Zugang zum NetBird-Netz haben.

Dokumentation: [Caddy Automatic HTTPS](https://caddyserver.com/docs/automatic-https),
[Caddy `tls` und interne CA](https://caddyserver.com/docs/caddyfile/directives/tls),
[NetBird Reverse Proxy](https://docs.netbird.io/manage/reverse-proxy).
