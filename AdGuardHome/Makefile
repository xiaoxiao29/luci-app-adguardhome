#
# Copyright (C) 2015-2016 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE.txt for more information.
#

include $(TOPDIR)/rules.mk

PKG_NAME:=AdGuardHome
PKG_VERSION:=0.107.79
PKG_RELEASE:=1

# 上游自 v0.107 起为“单二进制 + go:embed 前端”：前端由 client_v2 (npm/webpack)
# 构建到源码根的 build/ 目录，再由 main.go 的 //go:embed build 打包进二进制。
# 这里使用官方 codeload 源码包（比 git clone 快，且可用 PKG_HASH 校验收包）。
PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz
PKG_SOURCE_URL:=https://codeload.github.com/AdguardTeam/AdGuardHome/tar.gz/v$(PKG_VERSION)?
PKG_HASH:=5f4fa119f24ea741bb8cecfa74203165fe6b86b904cef05ff429fb227b07d425
PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

PKG_LICENSE:=GPL-3.0-only
PKG_LICENSE_FILES:=LICENSE.txt
PKG_MAINTAINER:=Yang Feng <yangfeng229@gmail.com>

PKG_CONFIG_DEPENDS:=CONFIG_ADGUARDHOME_COMPRESS_UPX

# ---------------------------------------------------------------------------
# 工具链说明
#
# 本包不使用 OpenWrt 的 golang/host 与 node/host：
#   * AdGuardHome 的 go.mod 要求 go >= 1.26.6，而 OpenWrt 24.10 的 golang feed
#     仍是 1.23.x；
#   * node/host 需要从源码编译 Node，代价极高。
# 因此直接调用外部工具链（默认优先取容器内预装的 ~/.openwrt/tools），
# 可用下列变量覆盖：
#   ADGH_GO              Go 可执行文件（需 >= 1.26）
#   ADGH_NODE_DIR        含 node/npm 的目录
#   ADGH_NPM             npm 可执行文件
#   ADGH_NPM_REGISTRY    npm 源（默认 npmmirror，国内更快）
#   ADGH_GOPROXY         Go 模块代理（默认 goproxy.cn）
# 工具链准备方式见 .devcontainer/README.md。
# ---------------------------------------------------------------------------
ADGH_TOOLS_DIR ?= $(HOME)/.openwrt/tools
ADGH_GO ?= $(if $(wildcard $(ADGH_TOOLS_DIR)/go/bin/go),$(ADGH_TOOLS_DIR)/go/bin/go,go)
ADGH_NODE_DIR ?= $(if $(wildcard $(ADGH_TOOLS_DIR)/node/bin/node),$(ADGH_TOOLS_DIR)/node/bin,$(if $(wildcard /usr/bin/node),/usr/bin,/usr/local/bin))
ADGH_NPM ?= $(ADGH_NODE_DIR)/npm
ADGH_NPM_REGISTRY ?= https://registry.npmmirror.com
ADGH_GOPROXY ?= https://goproxy.cn,direct

# OpenWrt ARCH -> GOARCH：纯 Go 交叉编译（CGO_ENABLED=0），无需 target 工具链
ifeq ($(ARCH),x86_64)
  ADGH_GOARCH:=amd64
else ifeq ($(ARCH),i386)
  ADGH_GOARCH:=386
else ifneq ($(filter aarch64%,$(ARCH)),)
  ADGH_GOARCH:=arm64
else ifneq ($(filter arm%,$(ARCH)),)
  ADGH_GOARCH:=arm
  ADGH_GOARM:=$(if $(findstring _v5,$(ARCH)),5,$(if $(findstring _v6,$(ARCH)),6,7))
else ifneq ($(filter mips64el%,$(ARCH)),)
  ADGH_GOARCH:=mips64le
  ADGH_GOMIPS:=softfloat
else ifneq ($(filter mipsel%,$(ARCH)),)
  ADGH_GOARCH:=mipsle
  ADGH_GOMIPS:=softfloat
else ifneq ($(filter mips64%,$(ARCH)),)
  ADGH_GOARCH:=mips64
  ADGH_GOMIPS:=softfloat
else ifneq ($(filter mips%,$(ARCH)),)
  ADGH_GOARCH:=mips
  ADGH_GOMIPS:=softfloat
else ifneq ($(filter riscv64%,$(ARCH)),)
  ADGH_GOARCH:=riscv64
else ifneq ($(filter powerpc64%,$(ARCH)),)
  ADGH_GOARCH:=ppc64
else ifneq ($(filter powerpc%,$(ARCH)),)
  ADGH_GOARCH:=ppc
else
  ADGH_GOARCH:=unsupported
endif

ADGH_VERSION_PKG:=github.com/AdguardTeam/AdGuardHome/internal/version
ADGH_LDFLAGS:=-s -w \
	-X $(ADGH_VERSION_PKG).version=v$(PKG_VERSION) \
	-X $(ADGH_VERSION_PKG).channel=release \
	-X $(ADGH_VERSION_PKG).committime=$(if $(SOURCE_DATE_EPOCH),$(SOURCE_DATE_EPOCH),0) \
	$(if $(ADGH_GOARM),-X $(ADGH_VERSION_PKG).goarm=$(ADGH_GOARM)) \
	$(if $(ADGH_GOMIPS),-X $(ADGH_VERSION_PKG).gomips=$(ADGH_GOMIPS))

include $(INCLUDE_DIR)/package.mk

define Package/AdGuardHome
  SECTION:=net
  CATEGORY:=Network
  TITLE:=Network-wide ads and trackers blocking DNS server
  URL:=https://github.com/AdguardTeam/AdGuardHome
  DEPENDS:=+ca-bundle
endef

define Package/AdGuardHome/description
  Free and open source, powerful network-wide ads and trackers blocking DNS server.
endef

define Package/$(PKG_NAME)/config
config ADGUARDHOME_COMPRESS_UPX
	bool "Compress executable files with UPX"
	default n
endef

define Build/Compile
	@if ! $(ADGH_GO) version 2>/dev/null | grep -qE 'go1\.(2[6-9]|[3-9][0-9])'; then \
		echo "ERROR: AdGuardHome $(PKG_VERSION) 需要 Go >= 1.26（go.mod 要求 1.26.6）"; \
		echo "       当前工具链: $$($(ADGH_GO) version 2>&1)"; \
		echo "       可用 ADGH_GO=/path/to/go 指定，准备方法见 .devcontainer/README.md"; \
		exit 1; \
	fi
	@echo "AdGuardHome: 构建前端 client_v2 (npm)"
	cd $(PKG_BUILD_DIR)/client_v2 && \
		PATH="$(ADGH_NODE_DIR):$(PATH)" $(ADGH_NPM) ci \
			--registry=$(ADGH_NPM_REGISTRY) --no-audit --no-fund && \
		PATH="$(ADGH_NODE_DIR):$(PATH)" $(ADGH_NPM) run build-prod
	@echo "AdGuardHome: 编译后端 (GOARCH=$(ADGH_GOARCH))"
	cd $(PKG_BUILD_DIR) && \
		CGO_ENABLED=0 GOOS=linux GOARCH=$(ADGH_GOARCH) \
		GOARM=$(ADGH_GOARM) GOMIPS=$(ADGH_GOMIPS) \
		GOPROXY=$(ADGH_GOPROXY) GOTOOLCHAIN=local \
		$(ADGH_GO) build -trimpath -ldflags "$(ADGH_LDFLAGS)" \
			-o $(PKG_BUILD_DIR)/AdGuardHome .
ifeq ($(CONFIG_ADGUARDHOME_COMPRESS_UPX),y)
	@if [ -x $(STAGING_DIR_HOST)/bin/upx ]; then \
		$(STAGING_DIR_HOST)/bin/upx --lzma --best $(PKG_BUILD_DIR)/AdGuardHome; \
	else \
		echo "WARNING: upx 不可用，跳过压缩"; \
	fi
endif
endef

# luci 插件默认使用 /etc/config/adGuardConfig/AdGuardHome（uci: binpath），
# 因此二进制装到该路径；/usr/bin/AdGuardHome 以符号链接提供命令行入口，
# 避免在 ipk 里重复打包 40MB 的二进制。
define Package/AdGuardHome/install
	$(INSTALL_DIR) $(1)/etc/config/adGuardConfig
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/AdGuardHome $(1)/etc/config/adGuardConfig/AdGuardHome
	$(INSTALL_DIR) $(1)/usr/bin
	$(LN) /etc/config/adGuardConfig/AdGuardHome $(1)/usr/bin/AdGuardHome
endef

$(eval $(call BuildPackage,AdGuardHome))
