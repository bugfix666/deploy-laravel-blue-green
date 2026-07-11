.DEFAULT_GOAL := help
SHELL = /bin/bash

# Эти переменные передаются из GitHub Actions через action
export ENV
export PORT
export IMAGE

BLUE_PORT = 8081
GREEN_PORT = 8082
CADDY_CONFIG = /www/Caddyfile

# Вспомогательные переменные (при локальном запуске можно задать вручную)
USER_ID=$(shell id -u)
GROUP_ID=$(shell id -g)
export USER_ID
export GROUP_ID

.PHONY: help deploy down switch-to-blue switch-to-green health-check

help:
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

deploy: ## Запустить контейнеры с новым образом, выполнить миграции и оптимизацию
	@echo "Deploying $(ENV) with image $(IMAGE) on port $(PORT)"
	docker compose -f docker-compose.prod.yml -p $(ENV) pull app
	docker compose -f docker-compose.prod.yml -p $(ENV) up -d --remove-orphans --force-recreate
	docker cp /www/${ENV}/.env larasuperdock-app-${ENV}:/www/.env
	docker exec larasuperdock-app-${ENV} php artisan config:clear
	@sleep 10
	@echo "=== Logs of larasuperdock-app-$(ENV) ==="
	@docker logs larasuperdock-app-$(ENV) --tail 50 || true
	@echo "Checking if port $(PORT) is open..."
	@nc -zv localhost $(PORT) || { echo "Port $(PORT) is not open"; exit 1; }
	@echo "Checking web server inside container..."
	@docker exec larasuperdock-app-$(ENV) curl -f -s -o /dev/null -w "%{http_code}" http://localhost:80/health || { echo "Web server inside container is not responding"; exit 1; }
	docker compose -f docker-compose.prod.yml -p $(ENV) exec -T app php artisan migrate --force
	docker compose -f docker-compose.prod.yml -p $(ENV) exec -T app php artisan optimize:clear

down: ## Остановить контейнеры текущей среды
	@docker compose -f docker-compose.prod.yml -p $(ENV) down || true

health-check: ## Проверить здоровье через /health (с повторными попытками)
	@echo "Checking health of $(ENV)..."
	@for i in 1 2 3 4 5; do \
		echo "Attempt $$i/5..."; \
		if curl -f --connect-timeout 5 --max-time 10 http://localhost:$(PORT)/health; then \
			echo "Health check passed"; \
			exit 0; \
		else \
			echo "Health check failed (attempt $$i), waiting 5 seconds..."; \
			sleep 5; \
		fi; \
	done; \
	echo "Health check failed after 5 attempts"; \
	docker logs larasuperdock-app-$(ENV) --tail 50; \
	exit 1

switch-to-blue: ## Переключить Caddy на blue
	@echo "Switching to blue"
	@printf ':80 {\n    encode zstd gzip\n    header {\n        -Server\n        X-Frame-Options "SAMEORIGIN"\n        X-Content-Type-Options "nosniff"\n        Referrer-Policy "strict-origin-when-cross-origin"\n        Permissions-Policy "geolocation=(), microphone=(), camera=()"\n    }\n    reverse_proxy localhost:$(BLUE_PORT)\n}\n' > $(CADDY_CONFIG)
	@sudo systemctl reload caddy || caddy reload --config $(CADDY_CONFIG) || true

switch-to-green: ## Переключить Caddy на green
	@echo "Switching to green"
	@printf ':80 {\n    encode zstd gzip\n    header {\n        -Server\n        X-Frame-Options "SAMEORIGIN"\n        X-Content-Type-Options "nosniff"\n        Referrer-Policy "strict-origin-when-cross-origin"\n        Permissions-Policy "geolocation=(), microphone=(), camera=()"\n    }\n    reverse_proxy localhost:$(GREEN_PORT)\n}\n' > $(CADDY_CONFIG)
	@sudo systemctl reload caddy || caddy reload --config $(CADDY_CONFIG) || true