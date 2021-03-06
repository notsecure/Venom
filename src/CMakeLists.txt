#
#    Copyright (C) 2013-2014 Venom authors and contributors
#
#    This file is part of Venom.
#
#    Venom is free software: you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation, either version 3 of the License, or
#    (at your option) any later version.
#
#    Venom is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#
#    You should have received a copy of the GNU General Public License
#    along with Venom.  If not, see <http://www.gnu.org/licenses/>.
#

# Required cmake modules
INCLUDE(FindVala)
INCLUDE(${VALA_USE_FILE})
INCLUDE(GResource)
INCLUDE(PkgCheckModulesFullpath)

# Validate version and availability through the previously defined modules
FIND_PACKAGE(Vala "0.18.1" REQUIRED)
FIND_PACKAGE(PkgConfig REQUIRED)

# Get dependencies through pkg-config
PKG_CHECK_MODULES_FULLPATH(GTK    REQUIRED gtk+-3.0>=3.4)
PKG_CHECK_MODULES_FULLPATH(GIO    REQUIRED gio-2.0>=2.32)
PKG_CHECK_MODULES_FULLPATH(GLIB   REQUIRED glib-2.0>=2.32)
PKG_CHECK_MODULES_FULLPATH(SQLITE REQUIRED sqlite3>=3.7)
PKG_CHECK_MODULES_FULLPATH(JSON   REQUIRED json-glib-1.0>=0.14)
PKG_CHECK_MODULES_FULLPATH(TOX    REQUIRED libtoxcore>=0.0)

# use djbdns as substitute for missing dns_txt in glib 2.32
IF(${GLIB_VERSION_MINOR} LESS 34)
  SET(ENABLE_DJBDNS ON)
ENDIF()

IF(${ENABLE_QR_ENCODE})
  PKG_CHECK_MODULES_FULLPATH(QR_ENCODE REQUIRED libqrencode>=3.1.1)
  SET(VENOM_CFLAGS_OPT ${VENOM_CFLAGS_OPT} ${QR_ENCODE_CFLAGS} ${QR_ENCODE_CFLAGS_OTHER})
  SET(VENOM_VALA_FLAGS_OPT ${VENOM_VALA_FLAGS_OPT} -D ENABLE_QR_ENCODE)
  SET(VENOM_LIBRARIES_OPT ${VENOM_LIBRARIES_OPT} ${QR_ENCODE_LIBRARIES})
ENDIF()

IF(${ENABLE_DJBDNS})
  INCLUDE(FindDJBDns)
  SET(VENOM_CFLAGS_OPT ${VENOM_CFLAGS_OPT} ${DJBDNS_CFLAGS})
  SET(VENOM_VALA_FLAGS_OPT ${VENOM_VALA_FLAGS_OPT} -D ENABLE_DJBDNS)
  SET(VENOM_LIBRARIES_OPT ${VENOM_LIBRARIES_OPT} ${DJBDNS_LIBRARIES})
ENDIF()

# Set a variable defining our minimal version of glib
SET(TARGET_GLIB ${GLIB_VERSION_MAJOR}.${GLIB_VERSION_MINOR})

# Workaround for libtoxcore and libsodium not providing info on
# all needed static libraries in their pkg-config files
# see https://github.com/irungentoo/ProjectTox-Core/issues/705
IF(LINK_TOX_STATIC)
  IF(WIN32)
    SET(TOX_LIBRARIES ${TOX_LIBRARIES} m.a ws2_32.a)
  ELSE(WIN32)
    SET(TOX_LIBRARIES ${TOX_LIBRARIES} m.so)
  ENDIF(WIN32)
ENDIF(LINK_TOX_STATIC)
IF(LINK_GTK_STATIC)
  IF(WIN32)
    SET(GTK_LIBRARIES ${GTK_LIBRARIES} usp10.a iphlpapi.a)
  ENDIF(WIN32)
ENDIF(LINK_GTK_STATIC)
IF(LINK_GIO_STATIC)
  IF(WIN32)
    SET(GIO_LIBRARIES ${GIO_LIBRARIES} dnsapi.a iconv.a)
  ENDIF(WIN32)
ENDIF(LINK_GIO_STATIC)

# add cflags and cflags_other of all dependencies to our cflags
SET(CMAKE_C_FLAGS ${CMAKE_C_FLAGS}
                  ${GTK_CFLAGS}    ${GTK_CFLAGS_OTHER}
                  ${GIO_CFLAGS}    ${GIO_CFLAGS_OTHER}
                  ${GLIB_CFLAGS}   ${GLIB_CFLAGS_OTHER}
                  ${SQLITE_CFLAGS} ${SQLITE_CFLAGS_OTHER}
                  ${JSON_CFLAGS}   ${JSON_CFLAGS_OTHER}
                  ${TOX_CFLAGS}    ${TOX_CFLAGS_OTHER}
                  ${VENOM_CFLAGS_OPT}
)

# since pkg_check_modules returns semicolon
# separated *_c_flags, we have to remove those
STRING(REPLACE ";" " " CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")

# replace the annotated variables in config.h.in and copy it to config.h
CONFIGURE_FILE( config.h.in config.h )

# define our source files
SET( VENOM_SRC
  Main.vala
  core/Contact.vala
  core/ContactStorage.vala
  core/DhtNode.vala
  core/DhtNodeStorage.vala
  core/FileTransfer.vala
  core/GroupChat.vala
  core/GroupChatContact.vala
  core/MessageLog.vala
  core/Message.vala
  core/ResourceFactory.vala
  core/Settings.vala
  core/SqliteTools.vala
  core/Tools.vala
  core/ToxDns.vala
  core/ToxSession.vala
  core/User.vala
  ui/AboutDialog.vala
  ui/AddContactDialog.vala
  ui/ChatMessage.vala
  ui/Client.vala
  ui/ContactFilter.vala
  ui/ContactListCellRenderer.vala
  ui/ContactListTreeView.vala
  ui/ContactListWindow.vala
  ui/ConversationTextView.vala
  ui/ConversationView.vala
  ui/ConversationWidget.vala
  ui/EditableLabel.vala
  ui/FileTransferChatEntry.vala
  ui/GroupConversationSidebar.vala
  ui/GroupConversationWidget.vala
  ui/MessageTextView.vala
  ui/PinDialog.vala
  ui/SearchEntry.vala
  ui/UserInfoWindow.vala
  ui/UITools.vala
  ui/SettingsWindow.vala
)

# precompile all vala code to c via valac
VALA_PRECOMPILE( VALA_C_VENOM
    ${VENOM_SRC}
  PACKAGES
    config
    djbdns
    gio-2.0
    glib-2.0
    gtk+-3.0
    json-glib-1.0
    libqrencode
    sqlite3
    tox-1.0
  OPTIONS
    --target-glib=${TARGET_GLIB}
    --thread
    --vapidir=${CMAKE_SOURCE_DIR}/src/vapi
    ${VENOM_VALA_FLAGS_OPT}
  CUSTOM_VAPIS
  GENERATE_VAPI
  GENERATE_HEADER
)

# compile glib resource files to c code
GLIB_COMPILE_RESOURCES( GLIB_RESOURCES_VENOM
  SOURCE
    ui/venom.gresource.xml
    pixmaps/venom_pixmaps.gresource.xml
)

# compile the main executable
ADD_EXECUTABLE( venom
  ${VALA_C_VENOM}
  ${GLIB_RESOURCES_VENOM}
  ${VENOM_RESOURCE_FILE}
)

# link the main executable to our dependencies
TARGET_LINK_LIBRARIES( venom
                       ${VENOM_LIBRARIES_OPT}
                       ${TOX_LIBRARIES}
                       ${SQLITE_LIBRARIES}
                       ${GIO_LIBRARIES}
                       ${GLIB_LIBRARIES}
                       ${GTK_LIBRARIES}
                       ${JSON_LIBRARIES}
)

INSTALL(TARGETS venom RUNTIME DESTINATION ${CMAKE_INSTALL_SYSTEM_RUNTIME_DESTINATION})

# add tests
ADD_SUBDIRECTORY(testing)

# Summary of optional features
MESSAGE("Optional features / dependencies:")
SET(DJBDNS_DESCRIPTION "ENABLE_DJBDNS - resolve tox dns ids using djbdns (required for glib <= 2.32)")
IF(${ENABLE_DJBDNS})
  MESSAGE("   [X] ${DJBDNS_DESCRIPTION}")
ELSE()
  MESSAGE("   [ ] ${DJBDNS_DESCRIPTION}")
ENDIF()
SET(QR_ENCODE_DESCRIPTION "ENABLE_QR_ENCODE - show QR codes (requires libqrencode)")
IF(${ENABLE_QR_ENCODE})
  MESSAGE("   [X] ${QR_ENCODE_DESCRIPTION}")
ELSE()
  MESSAGE("   [ ] ${QR_ENCODE_DESCRIPTION}")
ENDIF()
# vim:set ts=2 sw=2 et:
