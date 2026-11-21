# Nepřehlédni to: migrace z Vercelu

Soubory jsou součástí lokálního repozitáře Hnuti-kruh/neprehlednito.
Výchozí revize: 2308cf0b61331fdf8d5df97d6419c7d68361e4e5.
Soubor .github/workflows/ci.yml upravuje původní CI. Obsah webu se nemění.

### Navázání Git historie — provedeno 21. 11. 2026

Původní lokální main neměl commit ani remote. Všech 24 neignorovaných souborů
bylo porovnáno pomocí Git blob hashů se vzdáleným main; po Git normalizaci
nebyl žádný rozdíl ani chybějící či přidaný soubor. Po fetch byly index a lokální
main navázány na existující vzdálený commit, přidán origin a tracking origin/main.
Pracovní soubory se nepřepisovaly, žádný nový commit ani push nevznikl.
Tento postup by při rozdílných souborech vyžadoval samostatné zachování a sloučení změn.

## Build a image

- Multi-stage build: node:24.22.3-alpine3.24 + npm ci + astro build; runtime nginx:1.30.7-alpine3.24.
- Node, npm ani zdrojové soubory nejsou v runtime image.
- Runtime běží jako nginx na portu 8080, s read-only root filesystemem a tmpfs /tmp.
- CI sestaví a otestuje image na větvích a PR. Jen main (push nebo ruční workflow_dispatch na main) publikuje GHCR image.
- Publikuje se přesně otestovaná image, nikoli druhý samostatný build.
- Tagy: latest a sha-<celý commit SHA>. Pro nasazení používejte sha tag ze summary Actions, případně digest.
- Workflow nemění běžící kontejnery. Není potřeba vlastní GitHub secret pro publikování; používá GITHUB_TOKEN s packages:write. Organizace musí publikování balíčků povolovat.
- deployment/smoke-test.sh kontroluje Nginx konfiguraci, skutečný Docker healthy stav, non-root UID, hlavní stránku, cover.png, hashovaný CSS/JS asset, cache hlavičky, gzip a přesné HTTP 404 pro chybějící stránku i asset. Kontejner testuje se stejnými omezeními jako Compose a po testu jej odstraní.
- HTML a nehashované soubory používají no-cache (při opětovném použití validace); /_astro/ má roční immutable cache pouze pro nalezené assety. /healthz používá no-store.
- Statická image nevyžaduje produkční env ani secrets. Portainer proměnná WEB_IMAGE vybírá image; env kontejneru již nemění vygenerované HTML. Budoucí veřejná build konfigurace musí být výslovně oddělena od secrets.
- .dockerignore povoluje pouze build vstupy a Nginx konfiguraci, vylučuje env soubory, npm credentials a privátní klíče i v povolených adresářích. Žádné secrets se nepředávají jako build args.
- Základní image jsou připnuté verzí, nikoli digestem. Opakovaný build stejného commitu proto nemusí mít totožný digest; pro přesnou identitu nasazení použijte digest z publikace.

### Stav ověření — 21. 11. 2026

- npm ci a npm run build prošly lokálně (Node 22.18.0, Astro 5.13.5); následně prošel i build přes dočasný Node 24.22.3 shodný s Dockerfile. Jde o Windows build, nikoli ověření Linux image. Vzniklo statické dist/index.html a hashovaný CSS asset; package-lock.json se nemění.
- docker compose config --quiet, parsování YAML workflow/Compose, bash -n deployment/smoke-test.sh a git diff --check prošly. Kontrola výstupu našla hlavní HTML, cover.png a skutečný hashovaný CSS soubor (12 173 bajtů). GitHub Actions zatím spuštěny nebyly.
- Dostupnost obou linux/amd64 image ověřena přes docker manifest inspect bez daemonu. Všechny čtyři připnuté Action SHA odpovídaly uvedeným upstream major tagům přes git ls-remote.
- Docker daemon není dostupný. Docker build, nginx -t v kontejneru a runtime smoke testy zatím NEPROBĚHLY. První Actions musí projít před nasazením.

Zdroje pro kontrolu image: [Node](https://github.com/docker-library/official-images/blob/master/library/node),
[Nginx](https://github.com/docker-library/official-images/blob/master/library/nginx).
Postup publikace: [GitHub GHCR](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images),
[Docker test before push](https://docs.docker.com/build/ci/github-actions/test-before-push/).

### Lokální ověření po spuštění Dockeru

V kořeni projektu (Bash / Git Bash / WSL, Docker musí poskytovat Linux kontejnery):

```sh
docker build --pull --platform linux/amd64 -t neprehlednito:ci .
bash deployment/smoke-test.sh neprehlednito:ci
```

Smoke test používá náhodný volný loopback port, vyžaduje Docker CLI, Bash, curl a gzip.
Nepřipojuje se k VPS ani k produkčním službám. Matomo zůstává na matomo.czechnomad.cz.

## 1. GitHub

Commit a push se provedou až na výslovný pokyn vlastníka; zatím nejsou provedeny.
Před commitem zkontrolujte git diff a přidané soubory. Pracovní větev + PR je vhodná
cesta pro první kontrolu CI bez publikace; publikace je povolena pouze na main.
Počkejte na úspěšné Actions a zkopírujte image reference z jejich summary.
GHCR balíček může po prvním publikování vzniknout jako private. Buď jej vědomě přepněte na public, nebo v Portaineru přidejte GHCR registry s účtem oprávněným k balíčku a classic PAT s read:packages. Token nevkládejte do repozitáře ani do tohoto compose.

## 2. Portainer

Na serveru vmi3079089 založte stack neprehlednito.
Vložte deployment/compose.yml a nastavte stack environment variable:

    WEB_IMAGE=ghcr.io/hnuti-kruh/neprehlednito:sha-<celý SHA z Actions>

Nepoužívejte doslovný placeholder. Pro private image vyberte příslušnou GHCR registry.
Deploy stack. Nepotřebuje volumes ani síť backend. Loopback port 18080 slouží pro test před DNS přepnutím; ověřte, že je volný.
Použijte běžný Docker Standalone stack, nikoli Swarm. Ověřte existenci externí sítě proxy
a nepřítomnost jiné služby se stejným aliasem neprehlednito-web.

## 3. Test bez změny DNS

Na cílovém serveru:

    curl -fsS http://127.0.0.1:18080/healthz
    curl -fsS http://127.0.0.1:18080/ -o /tmp/neprehlednito-index.html
    grep -q 'Nepřehlédni' /tmp/neprehlednito-index.html
    curl -I http://127.0.0.1:18080/cover.png
    test "$(curl -sS -o /dev/null -w '%{http_code}' http://127.0.0.1:18080/nonexistent-smoke-test)" = 404

V Portaineru ověřte healthy stav kontejneru. V HTML najděte skutečnou /_astro/*.css
cestu a ověřte i ji pomocí curl -I (200, Cache-Control: public, max-age=31536000, immutable).
Po nasazení lze loopback publikaci ponechat pro diagnostiku nebo odstranit ze stacku;
Caddy službu oslovuje přímo v síti proxy.

Z Windows lze web prohlédnout přes SSH tunel (SERVER nahraďte skutečnou adresou nebo SSH aliasem):

    ssh -N -L 18080:127.0.0.1:18080 nomad@SERVER

Potom otevřete http://localhost:18080. Odkazy na hlavní web a analytika stále používají stávající služby.
Ověřte i mobilní zobrazení, obrázky, styly a consent banner.

## 4. Caddy a DNS — až po úspěšné kontrole

Před změnou zálohujte Caddyfile (obsahuje autentizační hash; zálohu nesdílejte):

    cp -a /opt/stacks/caddy/Caddyfile /opt/stacks/caddy/Caddyfile.before-neprehlednito-$(date +%Y%m%d-%H%M%S)

Na konec existujícího /opt/stacks/caddy/Caddyfile přidejte obsah deployment/Caddyfile.fragment.
Nespouštějte celý fragment jako náhradu konfigurace. Existující hosty musí zůstat zachované.
Validace a reload v existujícím kontejneru (bez recreate):

    docker exec caddy-caddy-1 caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
    # Pouze pokud předchozí příkaz skončil úspěšně:
    docker exec caddy-caddy-1 caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile

Reload proveďte pouze po úspěšné validaci.
Při úpravě Caddyfile zachovejte stejný soubor/inode (bind mount na soubor může po atomické
náhradě dál ukazovat na starý inode). Validujte konfiguraci přímo v kontejneru, nikoli
jen soubor na hostu. Po validaci a reloadu zkontrolujte log Caddy bez sdílení autentizačního hashe.
Před DNS přepnutím zaznamenejte původní A/AAAA/CNAME a snižte TTL s předstihem dle původního TTL.
Zkontrolujte případné CAA a CDN/proxy nastavení. Změňte pouze webové záznamy apex/www; ostatní DNS zachovejte.
Pro apex i www nastavte správné směrování na cílový server; starý AAAA nesmí posílat návštěvníky na jiný server.
Caddy získá veřejný certifikát po správném DNS směrování a dostupnosti 80/443.
Ověřte HTTPS, www -> apex redirect, statické assety a chování neexistující URL. Vercel projekt i domény zatím ponechte pro rollback.

## Aktualizace a návrat

Po úspěšném Actions změňte WEB_IMAGE v Portaineru na nový sha tag a aktualizujte stack s pull nové image.
Pro návrat aplikace nastavte předchozí WEB_IMAGE. Předchozí image v GHCR nemažte.
Pro návrat hostingu obnovte původní DNS a ověřte Vercel; respektujte DNS cache.
Při vrácení Caddy konfigurace přepište obsah existujícího souboru z konkrétní uložené
zálohy, znovu proveďte validate a při úspěchu reload. Zálohu ani Caddyfile nevkládejte do Git.
Statický web nepřijímá lokální data, takže nemá DB synchronizaci pro rollback.
Stávající služby Caddy, MariaDB a NocoDB se při nasazení webového stacku nerecreatují.
