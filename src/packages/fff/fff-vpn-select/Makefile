include $(TOPDIR)/rules.mk

PKG_NAME:=fff-vpn-select
PKG_RELEASE:=4

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)

include $(INCLUDE_DIR)/package.mk

define Package/$(PKG_NAME)
	SECTION:=base
	CATEGORY:=Freifunk
	TITLE:= Freifunk-Franken vpn-select
	URL:=http://www.freifunk-franken.de
	DEPENDS:=+fff-fastd
endef

define Package/$(PKG_NAME)/description
	This package selects and starts the VPN
	(currently only via fastd)
endef

define Build/Prepare
	echo "all: " > $(PKG_BUILD_DIR)/Makefile
endef

define Build/Configure
	# nothing
endef

define Build/Compile
	# nothing
endef

define Package/$(PKG_NAME)/install
	$(CP) ./files/* $(1)/
endef

$(eval $(call BuildPackage,$(PKG_NAME)))
