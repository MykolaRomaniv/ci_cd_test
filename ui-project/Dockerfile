# Build the web app

FROM node:24-alpine

WORKDIR /app

RUN npm install pnpm -g

# Copy the dependency list (package.json and pnpm-lock.yaml)
COPY ./package.json pnpm-lock.yaml /app/

# In case we need to use a private npm registry
# COPY ./.npmrc /app/

# Copy the rest of the app
COPY . /app/

# Install dependencies
RUN pnpm install --frozen-lockfile
CMD ["pnpm", "run", "start"]