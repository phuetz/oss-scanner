# Construit Code Buddy depuis son dépôt (Node 22, comme .nvmrc), puis lance les tests de sécurité.
FROM node:22-bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 build-essential bash ripgrep procps \
    && rm -rf /var/lib/apt/lists/*

# contrat du scanner : la copie du dépôt vit dans l'image, sous /src
COPY . /src
WORKDIR /src

# Toutes les dépendances sont récupérées ici (le scan se fait ensuite sans réseau).
RUN npm ci --no-audit --no-fund
RUN npm run build
RUN npm link

# Tests de sécurité rapides pour vérifier l'image, sans faire échouer la construction.
# La suite complète (~27 000 tests) reste disponible : npx vitest run tests/<chemin>.
ENV HOME=/tmp/cb-home CODEBUDDY_TELEMETRY=false
RUN mkdir -p /tmp/cb-home && (npx vitest run tests/security tests/hooks --reporter=dot || echo "WARNING: vitest reported failures")
