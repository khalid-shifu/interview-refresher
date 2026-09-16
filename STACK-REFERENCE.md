# Stack Recognition Reference (Docker Exam Prep)

**The goal of this file:** when handed an unfamiliar project folder with no instructions, be able
to answer — "what stack is this, how do I install its dependencies, how do I build it, and how do
I start it?" — just by looking at which files are present. That's everything needed to write a
correct Dockerfile.

**The workflow to follow every time, in order:**

```
1. What files are sitting in this project's root?
        ↓
2. Which ecosystem do they belong to? (Node? Java? Python? ...)
        ↓
3. Which specific tool inside that ecosystem? (npm vs Yarn? Maven vs Gradle? ...)
        ↓
4. Look up: install command, build command, start command (tables below)
        ↓
5. If a package.json / build.gradle / pom.xml has a "scripts" or task list — check it.
   That's the real source of truth, more reliable than memorized defaults.
        ↓
6. Turn that into: FROM, WORKDIR, COPY, RUN (install + build), EXPOSE, CMD/ENTRYPOINT
```

Backend tables are ordered **most-likely-to-appear first** (⭐⭐⭐ = very common in training/exam
material, ⭐ = less common, good to recognize but lower priority to memorize deeply).

---

## Backend

| Priority | Stack | Recognize by (files present) | Install / dependency command | Build command | Start / run command |
|---|---|---|---|---|---|
| ⭐⭐⭐ | Java + Maven | `pom.xml`, `mvnw`, `mvnw.cmd` | (bundled into build step) | `mvn clean package` (or `./mvnw clean package` if a wrapper is present — always prefer the wrapper, it pins the exact Maven version) | `java -jar target/*.jar` — see wildcard note below, no need to know the exact jar name |
| ⭐⭐⭐ | Java + Gradle | `build.gradle` or `build.gradle.kts` (Kotlin DSL — still Gradle), `gradlew`, `gradlew.bat`, `settings.gradle` | (bundled into build step) | `./gradlew build` (or `./gradlew bootJar` for Spring Boot specifically) | `java -jar build/libs/*.jar` — see wildcard note below, no need to know the exact jar name |
| ⭐⭐⭐ | Node.js + npm | `package.json` + `package-lock.json` | `npm install` (or `npm ci` in Docker — installs exactly what the lock file pins, no surprises) | `npm run build` (only if the project has a build step — check `package.json` → `"scripts"` first) | `npm start` (defined under `package.json` → `"scripts"` → `"start"` — check it, don't assume) |
| ⭐⭐⭐ | Node.js + Yarn | `package.json` + `yarn.lock` | `yarn install` | `yarn build` | `yarn start` |
| ⭐⭐ | Node.js + pnpm | `package.json` + `pnpm-lock.yaml` | `pnpm install` | `pnpm build` | `pnpm start` |
| ⭐ | Node.js + Bun | `package.json` + `bun.lock`/`bun.lockb` | `bun install` | `bun run build` | `bun start` |
| ⭐⭐ | Python (plain) | `requirements.txt` | `pip install -r requirements.txt` | — (Python isn't compiled; there's usually no separate build step) | Varies — check for a `main.py`, `app.py`, `manage.py`, etc. |
| ⭐⭐ | Python + Django | `manage.py` + (`requirements.txt` or `pyproject.toml`) | same as whichever Python dependency tool is present | — | Dev: `python manage.py runserver 0.0.0.0:8000`. Production: usually `gunicorn <project>.wsgi:application --bind 0.0.0.0:8000` |
| ⭐⭐ | Python + FastAPI | `main.py` (or similar entrypoint) + (`requirements.txt` or `pyproject.toml`) | same as above | — | `uvicorn main:app --host 0.0.0.0 --port 8000` (module name before the colon must match the actual Python file) |
| ⭐ | Python + Poetry | `pyproject.toml` + `poetry.lock` | `poetry install` | — | `poetry run python <entry>.py` (or whatever the project defines) |
| ⭐ | Python + Pipenv | `Pipfile` + `Pipfile.lock` | `pipenv install` | — | `pipenv run python <entry>.py` |
| ⭐ | Python + uv | `pyproject.toml` + `uv.lock` | `uv sync` | — | `uv run python <entry>.py` |
| ⭐ | .NET | `*.csproj`, `*.sln` | `dotnet restore` | `dotnet publish -c Release -o /app/publish` | `dotnet <AppName>.dll` |
| ⭐ | Go | `go.mod`, `go.sum` | `go mod download` | `go build -o app` | `./app` — note: this produces a real compiled binary, so the final image can be a minimal multi-stage build (heavy Go toolchain in the build stage, tiny base + one binary in the final stage) |
| ⭐ | PHP + Composer | `composer.json`, `composer.lock` | `composer install` | — | Typically served via **PHP-FPM + Nginx**, not `php artisan serve` (that's dev-only). If `artisan` is present, it's Laravel specifically. |
| ⭐ | Ruby | `Gemfile`, `Gemfile.lock` | `bundle install` | — | `bundle exec puma` (Rails production) or `ruby app.rb` for a plain script |
| ⭐ | Rust | `Cargo.toml`, `Cargo.lock` | (bundled into build step) | `cargo build --release` | `./target/release/<binary-name>` — also a compiled binary, same multi-stage logic as Go |

> **Wildcard trick — never memorize or guess the exact jar filename (Maven/Gradle).** The jar's
> real name comes from build config (`rootProject.name` + `version` in
> `settings.gradle`/`build.gradle` for Gradle, or `<artifactId>` + `<version>` in `pom.xml` for
> Maven) — it's rarely a clean, guessable name. Instead of trying to recall it, **rename it on the
> way into the image** using a wildcard `COPY`, so the `CMD`/`ENTRYPOINT` line becomes fixed and
> never needs to know the original name:
> ```dockerfile
> FROM openjdk:17-jre-slim
> WORKDIR /app
> COPY target/*.jar app.jar          # Maven — or build/libs/*.jar for Gradle
> CMD ["java", "-jar", "app.jar"]
> ```
> Since there's normally only one jar sitting in that output folder after a build, `*.jar` matches
> it regardless of what it's actually called. If the *exact* name is ever needed for another
> reason, the reliable way to check is just looking: `ls target/*.jar` (Maven) or
> `ls build/libs/*.jar` (Gradle) — more trustworthy under exam pressure than deriving it from
> memory.

---

## Frontend

**Important distinction before the table:** most frontend frameworks produce **static files**
(HTML/CSS/JS) that need to be *served*, not *run* — so their Dockerfile pattern is usually
multi-stage: build the static files in a Node stage, then hand them to Nginx in a second, much
smaller stage. **Next.js is the one common exception** — it can run a live Node server instead of
producing pure static output.

```dockerfile
# Typical frontend pattern (React/Vue/Angular/Svelte — NOT Next.js)
FROM node:20 AS build
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

Compare that to a typical **backend** Node Dockerfile — same base image, completely different
ending:

```dockerfile
# Typical backend Node pattern — no second stage, no Nginx, Node itself stays running
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]
```

The difference to internalize: **backend Node = Node keeps running as the server. Frontend
React/Vue/Angular = Node's job ends after producing static files; something else (Nginx) serves
them from then on.**

| Priority | Stack | Recognize by (files present) | Install | Build command | Static output folder | How it's served |
|---|---|---|---|---|---|---|
| ⭐⭐⭐ | React + Vite | `package.json` + `vite.config.js`/`.ts` | `npm install` | `npm run build` | `dist/` | Nginx (multi-stage) |
| ⭐⭐⭐ | React (Create React App) | `package.json` + `src/`, **no** vite config | `npm install` | `npm run build` | `build/` | Nginx (multi-stage) |
| ⭐⭐ | Angular | `angular.json` | `npm install` | `npm run build` or `ng build` | `dist/<project-name>/` (nested one level deeper than the others — easy to trip on) | Nginx (multi-stage) |
| ⭐⭐ | Vue | `package.json` + `vite.config.js`/`.ts` (or `vue.config.js` for older Vue-CLI projects) | `npm install` | `npm run build` | `dist/` | Nginx (multi-stage) |
| ⭐ | Next.js | `next.config.js`/`.ts` + `package.json` | `npm install` | `npm run build` | `.next/` — **not plain static files** | A live Node server: `npm start` (or the standalone output pattern already covered in section 4 of the main Docker README) |
| ⭐ | Svelte / SvelteKit | `svelte.config.js` | `npm install` | `npm run build` | `build/` or `.svelte-kit/` (depends on the adapter used) | Nginx if static adapter, Node if the Node adapter is used — check `svelte.config.js` |

---

## Sample Dockerfiles: Basic vs Advanced

For every stack above, two versions: a **Basic** one (single stage, `COPY . .` then install —
simplest thing that works, no optimization) and an **Advanced** one (dependency files copied in
*before* the rest of the source for layer caching, plus a multi-stage build wherever that's a
real, common pattern). These are patterns to recognize and reproduce, not exact copies — real
project filenames (jar names, `.dll` names, Angular's nested output folder, etc.) will differ.

### Java + Maven

**Basic:**
```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY . .
RUN chmod +x ./mvnw
RUN ./mvnw clean package
CMD java -jar target/*.jar
```

**Advanced:**
```dockerfile
FROM openjdk:17-jdk-slim AS build
WORKDIR /app
COPY mvnw .
COPY .mvn .mvn/
COPY pom.xml .
RUN chmod +x ./mvnw
RUN ./mvnw dependency:go-offline
COPY src ./src
RUN ./mvnw clean package

FROM openjdk:17-jre-slim
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```
> Uses `./mvnw` (the project's own Maven wrapper), same reasoning as Gradle's `./gradlew` below —
> it pins the exact Maven version the project expects, so no Maven install step is needed in the
> image at all. `pom.xml` is copied and dependencies resolved *before* `src/` is copied in — so
> editing application code alone doesn't force Maven to re-download every dependency on the next
> build.

### Java + Gradle

**Basic:**
```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY . .
RUN chmod +x ./gradlew
RUN ./gradlew build
CMD java -jar build/libs/*.jar
```

**Advanced:**
```dockerfile
FROM openjdk:17-jdk-slim AS build
WORKDIR /app
COPY gradlew .
COPY gradle gradle/
COPY build.gradle settings.gradle ./
RUN chmod +x ./gradlew
RUN ./gradlew dependencies
COPY src ./src
RUN ./gradlew build

FROM openjdk:17-jre-slim
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```
> Uses `./gradlew` (the project's own wrapper script), not a bare `gradle` command — the wrapper
> downloads and uses the **exact** Gradle version the project expects, regardless of what's (or
> isn't) installed in the base image. That's why the base image here is a plain JDK image, not a
> `gradle:...` image — the wrapper brings its own Gradle, so nothing needs to be pre-installed.
> `chmod +x ./gradlew` is there because the execute bit on that script doesn't always survive a
> `COPY` (especially coming from a Windows checkout).

### Node.js + npm

**Basic:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]
```

**Advanced:**
```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:20-alpine
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```
> `package.json`/`package-lock.json` copied first, so `npm ci` is only re-run when dependencies
> actually change — not on every source code edit.

### Node.js + Yarn

**Basic:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN yarn install
EXPOSE 3000
CMD ["yarn", "start"]
```

**Advanced:**
```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

FROM node:20-alpine
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["yarn", "start"]
```

### Node.js + pnpm

**Basic:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
RUN npm install -g pnpm
COPY . .
RUN pnpm install
EXPOSE 3000
CMD ["pnpm", "start"]
```

**Advanced:**
```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
RUN npm install -g pnpm
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

FROM node:20-alpine
WORKDIR /app
RUN npm install -g pnpm
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["pnpm", "start"]
```

### Node.js + Bun

**Basic:**
```dockerfile
FROM oven/bun:1-alpine
WORKDIR /app
COPY . .
RUN bun install
EXPOSE 3000
CMD ["bun", "start"]
```

**Advanced:**
```dockerfile
FROM oven/bun:1-alpine AS deps
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile

FROM oven/bun:1-alpine
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["bun", "start"]
```

### Python (plain)

**Basic:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 8000
CMD ["python", "app.py"]
```

**Advanced:**
```dockerfile
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=build /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
EXPOSE 8000
CMD ["python", "app.py"]
```
> `--user` installs packages into `/root/.local` instead of system-wide, which makes them easy to
> copy wholesale into a fresh final stage.

### Python + Django

**Basic:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

**Advanced:**
```dockerfile
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=build /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
EXPOSE 8000
CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]
```
> Notice the Basic version uses Django's dev server, while the Advanced/production version swaps
> in `gunicorn` — a separate, real distinction worth knowing on its own.

### Python + FastAPI

**Basic:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Advanced:**
```dockerfile
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=build /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Python + Poetry

**Basic:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install --no-cache-dir poetry
COPY . .
RUN poetry install --no-root
EXPOSE 8000
CMD ["poetry", "run", "python", "main.py"]
```

**Advanced:**
```dockerfile
FROM python:3.12-slim AS build
WORKDIR /app
RUN pip install --no-cache-dir poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt --output requirements.txt --without-hashes
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=build /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
EXPOSE 8000
CMD ["python", "main.py"]
```
> `poetry export` converts the lock file to a plain `requirements.txt` so the final stage can use
> a plain, lightweight `pip install`, without needing Poetry itself in the runtime image at all.

### Python + Pipenv

**Basic:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install --no-cache-dir pipenv
COPY . .
RUN pipenv install --system --deploy
EXPOSE 8000
CMD ["python", "main.py"]
```

**Advanced:**
```dockerfile
FROM python:3.12-slim AS build
WORKDIR /app
RUN pip install --no-cache-dir pipenv
COPY Pipfile Pipfile.lock ./
RUN pipenv requirements > requirements.txt
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=build /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
EXPOSE 8000
CMD ["python", "main.py"]
```

### Python + uv

**Basic:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY . .
RUN uv sync --frozen
EXPOSE 8000
CMD ["uv", "run", "python", "main.py"]
```

**Advanced:**
```dockerfile
FROM python:3.12-slim AS build
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project

FROM python:3.12-slim
WORKDIR /app
COPY --from=build /app/.venv /app/.venv
COPY . .
ENV PATH=/app/.venv/bin:$PATH
EXPOSE 8000
CMD ["python", "main.py"]
```

### .NET

**Basic:**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0
WORKDIR /app
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish
EXPOSE 8080
ENTRYPOINT ["dotnet", "/app/publish/MyApp.dll"]
```

**Advanced:**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /app
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```
> `MyApp.dll` has to match the actual project name — no wildcard shortcut for `.dll` files, so
> check `ls /app/publish/*.dll` or the `.csproj` filename if unsure.

### Go

**Basic:**
```dockerfile
FROM golang:1.22-alpine
WORKDIR /app
COPY . .
RUN go build -o app
EXPOSE 8080
CMD ["./app"]
```

**Advanced:**
```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o app

FROM alpine:latest
WORKDIR /app
COPY --from=build /app/app .
EXPOSE 8080
CMD ["./app"]
```
> Go's Advanced version is the clearest illustration of multi-stage builds paying off: the final
> image doesn't carry the Go compiler at all, only the compiled binary.

### PHP + Composer

**Basic:**
```dockerfile
FROM php:8.3-cli
WORKDIR /app
COPY . .
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
RUN composer install --no-dev
EXPOSE 8000
CMD ["php", "-S", "0.0.0.0:8000", "-t", "public"]
```

**Advanced:**
```dockerfile
FROM composer:2 AS build
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --ignore-platform-reqs
COPY . .

FROM php:8.3-cli
WORKDIR /app
COPY --from=build /app .
EXPOSE 8000
CMD ["php", "-S", "0.0.0.0:8000", "-t", "public"]
```
> Both still use PHP's built-in dev server for simplicity — real production PHP typically runs
> PHP-FPM behind Nginx, a longer two-process setup, but good enough to recognize and mention.

### Ruby

**Basic:**
```dockerfile
FROM ruby:3.3-slim
WORKDIR /app
COPY . .
RUN bundle install
EXPOSE 3000
CMD ["bundle", "exec", "puma", "-b", "tcp://0.0.0.0:3000"]
```

**Advanced:**
```dockerfile
FROM ruby:3.3-slim AS build
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install

FROM ruby:3.3-slim
WORKDIR /app
COPY --from=build /usr/local/bundle /usr/local/bundle
COPY . .
EXPOSE 3000
CMD ["bundle", "exec", "puma", "-b", "tcp://0.0.0.0:3000"]
```

### Rust

**Basic:**
```dockerfile
FROM rust:1.78
WORKDIR /app
COPY . .
RUN cargo build --release
EXPOSE 8080
CMD ["./target/release/myapp"]
```

**Advanced:**
```dockerfile
FROM rust:1.78 AS build
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
WORKDIR /app
COPY --from=build /app/target/release/myapp .
EXPOSE 8080
CMD ["./myapp"]
```
> `myapp` must match the `name` field under `[package]` in `Cargo.toml`. True dependency-layer
> caching for Rust needs an extra trick (a dummy `src/main.rs` built once with just
> `Cargo.toml`/`Cargo.lock` copied in, before the real source is copied over) — good to know it
> exists, rarely essential to memorize for an exam.

---

### Frontend — Basic vs Advanced

All frontend frameworks below (except Next.js) follow the same idea: the **Basic** version stays
single-stage and serves the build output with a simple Node-based static server (no Nginx, no
multi-stage — everything, including the Node build tools, ships in the final image). The
**Advanced** version is the real-world pattern: multi-stage, ending in a clean Nginx image with
nothing but the static files in it.

### React + Vite

**Basic:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "dist", "-l", "3000"]
```

**Advanced:**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

### React (Create React App)

**Basic:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "build", "-l", "3000"]
```

**Advanced:**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
```
> Only real difference from the Vite version: `build/` instead of `dist/` — CRA's output folder name.

### Angular

**Basic:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "dist/my-angular-app", "-l", "3000"]
```

**Advanced:**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist/my-angular-app /usr/share/nginx/html
EXPOSE 80
```
> `my-angular-app` is the project name from `angular.json` — Angular nests its output one folder
> deeper than the others (`dist/<project-name>/`), an easy detail to miss.

### Vue

**Basic:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "dist", "-l", "3000"]
```

**Advanced:**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

### Next.js

**Basic:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

**Advanced:**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/package.json ./
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/.next ./.next
COPY --from=build /app/public ./public
EXPOSE 3000
CMD ["npm", "start"]
```
> No Nginx here even in the Advanced version — Next.js keeps running as a live Node server, same
> as a backend Node app. The multi-stage benefit is still real though: dev-only files and the
> build cache from the `build` stage never make it into the final image.

### Svelte / SvelteKit (static adapter)

**Basic:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "build", "-l", "3000"]
```

**Advanced:**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
```
> Only true if `svelte.config.js` uses the **static** adapter. With the Node adapter, it follows
> the Next.js pattern instead (a live Node server, no Nginx, in both Basic and Advanced).

---

## Quick decision cheat-sheet, in the order to check things

1. **See a `package.json`?** → it's Node.js. Look for which lock file sits next to it
   (`package-lock.json` = npm, `yarn.lock` = Yarn, `pnpm-lock.yaml` = pnpm, `bun.lockb` = Bun) to
   know the exact install command.
2. **See `pom.xml`?** → Maven. **See `build.gradle` or `build.gradle.kts`?** → Gradle (the `.kts`
   variant is just Gradle's Kotlin syntax, same tool).
3. **See `requirements.txt`, `Pipfile`, or `pyproject.toml`?** → Python. Check which one specifically
   to know whether it's plain pip, Pipenv, Poetry, or uv.
4. **See `manage.py`?** → Django specifically, regardless of which Python dependency tool is used.
5. **See `*.csproj` or `*.sln`?** → .NET.
6. **See `go.mod`?** → Go.
7. **See `composer.json`?** → PHP. **Also see `artisan`?** → Laravel specifically.
8. **See `Gemfile`?** → Ruby (**also see `config/` + `app/` + `bin/rails`?** → Ruby on Rails specifically).
9. **See `Cargo.toml`?** → Rust.
10. **For frontend specifically:** `vite.config.*` → Vite-based (React or Vue). `angular.json` →
    Angular. `next.config.*` → Next.js. No config file but a `public/index.html` + `src/` → likely
    Create React App.
11. **Whenever unsure about the exact build/start command** — open `package.json`'s `"scripts"`
    section (Node), or check for a `Procfile`/`Makefile`/README fragment in the project. Written
    scripts in the project itself always beat memorized defaults.

---

*A companion spreadsheet with this same data (`stack-reference.csv`) is available for a quick
lookup table during the exam — open it in Excel/Google Sheets/LibreOffice Calc.*
