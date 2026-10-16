# Étape 1 : image de base versionnée
FROM node:24-slim

# Étape 2 : répertoire de travail
WORKDIR /app

# Étape 3 : copie des fichiers de dépendances
COPY package*.json ./

# Étape 4 : installation des dépendances de production
RUN npm install --production

# Étape 5 : copie du code source
COPY . .

# Étape 6 : variables par défaut
ENV PORT=3000
ENV HOST=0.0.0.0

# Étape 7 : démarrage
CMD ["npm", "start"]
