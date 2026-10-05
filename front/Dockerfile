# Étape 1 : build du front
FROM node:24-slim AS build

WORKDIR /app

# Copier les fichiers de dépendances
COPY package*.json ./

# Installer les dépendances
RUN npm install

# Copier le reste du code
COPY . .

# Build du front
RUN npm run build

# Étape 2 : serveur web nginx
FROM nginx:alpine

# Copier le build dans nginx
COPY --from=build /app/dist /usr/share/nginx/html

# Remplacer la configuration par défaut pour proxy /api vers le backend
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Exposer le port
EXPOSE 80

# Démarrage de nginx
CMD ["nginx", "-g", "daemon off;"]
