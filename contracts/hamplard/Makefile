# Hamplard Contract — Makefile

NETWORK       = testnet
CONTRACT_NAME = hamplard
WASM_PATH     = target/wasm32v1-none/release/$(CONTRACT_NAME).wasm
WASM_OPT      = target/wasm32v1-none/release/$(CONTRACT_NAME).optimized.wasm

.PHONY: test build optimize deploy-testnet clean fmt fmt-check check lint snapshots

test:
	cargo test

build:
	stellar contract build

optimize: build
	stellar contract optimize --wasm $(WASM_PATH)
	@echo "Optimized: $(WASM_OPT)"

deploy-testnet: optimize
	stellar contract deploy \
		--wasm $(WASM_OPT) \
		--source $(STELLAR_ACCOUNT) \
		--network $(NETWORK)

fmt:
	cargo fmt

fmt-check:
	cargo fmt --check

check:
	cargo check

lint:
	cargo clippy --all-targets -- -D warnings

snapsorts:
	cargo clean
	cargo test

clean:
	cargo clean
