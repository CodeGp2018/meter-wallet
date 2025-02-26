<template>
  <v-app v-resize="onResize" class="app-fadein">
    <!-- required by tab button -->
    <svg height="0" width="0">
      <defs>
        <clipPath id="left-corner">
          <path d="M5 0 h1 v5 h-6 a5 5 90 0 0 5 -5" />
        </clipPath>
        <clipPath id="right-corner">
          <path d="M1 0 a5 5 90 0 0 5 5 h-6" />
        </clipPath>
        <clipPath id="app-button-badge">
          <path
            d="M 0 0 h 14 a 7 7 90 0 1 7 7 v 14 a 3 3 -90 0 0 -3 -3 h -8 a 7 7 90 0 1 -7 -7 v -8 a 3 3 -90 0 0 -3 -3"
          />
        </clipPath>
      </defs>
    </svg>
    <DialogProxy :v-show="false" />
    <div class="toolbar">
      <v-layout class="drag">
        <transition-group
          tag="v-layout"
          class="tab-bar"
          @dblclick.native="onDblClickTitleBar"
          @mousedown.native.self.prevent
          name="tab-button"
        >
          <!-- here use @mouseup instead of @click, 
          since the area of window title is not responsive on osx-->
          <TabButton
            v-for="(page,i) in pages"
            class="tab-button no-drag"
            :key="'tab'+page.id"
            :title="page.title"
            :favicon="page.isBuiltin? '': page.favicon"
            :placeholder="page.isBuiltin? page.favicon : ''"
            :active="i===activePageIndex"
            @close="closeTab(i)"
            @mouseup.native="activePageIndex = i"
            @dblclick.native.stop
            @mousedown="i=== activePageIndex && $event.preventDefault()"
          />
          <v-btn
            class="no-drag ma-1 pa-0 ml-2"
            flat
            small
            key="the-new-tab-button"
            :ripple="false"
            @click="openTab('')"
            @dblclick.native.stop
            style="width:auto;height:auto;min-width:auto;"
          >
            <v-icon style="font-size:150%">add</v-icon>
          </v-btn>
        </transition-group>
        <WindowControls v-if="!isDarwin" class="no-drag" style="flex:0 0 auto;" />
      </v-layout>
      <div class="nav-bar">
        <v-layout row align-center px-1>
          <v-btn
            class="my-1"
            small
            icon
            :disabled="!activePage.canGoBack"
            :ripple="false"
            @click="activePage.goBack()"
          >
            <v-icon style="font-size:150%">arrow_back</v-icon>
          </v-btn>
          <v-btn
            class="my-1"
            small
            icon
            :disabled="!activePage.canGoForward"
            :ripple="false"
            @click="activePage.goForward()"
          >
            <v-icon style="font-size:150%">arrow_forward</v-icon>
          </v-btn>
          <v-btn
            class="my-1"
            small
            :ripple="false"
            icon
            :disabled="activePage.isBuiltin"
            @click="activePage.reloadOrStop($event.shiftKey)"
          >
            <v-icon style="font-size:150%">{{activePage.loading ? 'close': 'refresh'}}</v-icon>
          </v-btn>
          <div ref="navBox" class="mx-2 nav-box" :class="{'nav-box-focused': urlBoxFocused}">
            <v-layout align-center style="position:relative;" fill-height>
              <NodeStatusPanel :nudge-top="2">
                <v-btn
                  :ripple="false"
                  flat
                  slot="activator"
                  class="ma-0 px-1"
                  style="min-width:auto;"
                >
                  <NodeStatus />
                </v-btn>
              </NodeStatusPanel>
              <v-layout ref="urlBoxWithIcon" class="url-box-with-icon" fill-height align-center>
                <CertIndicator
                  v-if="!!activePage.cert && !activePage.userInput"
                  class="mx-2"
                  :cert="activePage.cert"
                />
                <v-icon v-else style="font-size:95%" class="mx-2">mdi-earth</v-icon>
                <AccessHistoryPanel
                  style="flex:1 1 auto;height:100%;transition: all 0s;"
                  v-model="showAccessHistory"
                  absolute
                  :close-on-click="false"
                  :close-on-content-click="false"
                  :position-x="accessHistoryPosition.x"
                  :position-y="accessHistoryPosition.y"
                  :width="accessHistoryPosition.width"
                  :keyword="keyword"
                  @update:selection="activePage.userInput=$event.href"
                  @select="onAccessHistorySelected"
                >
                  <UrlBox
                    @click.stop
                    slot="activator"
                    v-model="activePage.userInput"
                    :href="activePage.href"
                    @update:href="navigateTo"
                    class="url-box"
                    placeholder="Enter URL | app | block | tx | account to go"
                    @focus="urlBoxFocused=true"
                    @blur="onUrlBoxBlur"
                    @input="onUrlBoxInput"
                  />
                </AccessHistoryPanel>
              </v-layout>
              <ExpansionBtn
                v-show="showAddShortcutBtn"
                small
                flat
                style="min-width:auto;text-transform:none"
                :ripple="false"
                class="ma-0 caption"
                @action="addOrRemoveShortcut"
              >
                <v-icon
                  style="font-size:150%"
                >{{shortcutAdded ? 'mdi-bookmark-plus' : 'mdi-bookmark-plus-outline'}}</v-icon>
                <template slot="expansion">{{shortcutAdded?'Remove shortcut': 'Add shortcut'}}</template>
              </ExpansionBtn>
              <v-progress-linear
                v-for="(page,i) in pages"
                :key="'progress'+page.id"
                v-show="i===activePageIndex"
                background-color="rgba(0,0,0,0)"
                :active="!page.isBuiltin && page.loading"
                :value="page.progress"
                class="ma-0"
                height="2px"
                style="position:absolute;left:0;right:0;bottom:0;"
              />
            </v-layout>
          </div>
          <!--
          <ActivitiesPanel>
            <v-btn class="my-1" icon small slot="activator" :ripple="false" title="activities">
              <ActivitiesStatus />
            </v-btn>
          </ActivitiesPanel>
          -->

          <WindowMenu :items="menuItems">
            <v-btn class="my-1" icon small slot="activator" :ripple="false">
              <v-icon style="font-size:150%" title="menu">menu</v-icon>
            </v-btn>
          </WindowMenu>

          <ExperimentalMenu :items="experimentalItems">
            <v-btn class="my-1" icon small slot="activator" :ripple="false">
              <v-icon
                class="mdi mdi-layers-outline"
                style="font-size:150%"
                title="experimental features"
              />
            </v-btn>
          </ExperimentalMenu>
        </v-layout>
        <div class="sharp-line" />
      </div>
    </div>
    <v-content>
      <Vendor />
      <Swiper
        ref="swiper"
        style="width:100%;height:100%;"
        :canSwipeRight="activePage.canGoBack"
        :canSwipeLeft="activePage.canGoForward"
        @swipe="onSwipe"
      >
        <template v-for="(page,i) in pages">
          <Launcher
            v-if="page.isBuiltin"
            v-show="i===activePageIndex"
            :key="'launcher'+page.id"
            :href.sync="page.href"
            :nav="page.nav"
            @update:status="page.updateStatus($event)"
            style="position:absolute;left:0;top:0;right:0;bottom:0;"
            @update:href="page.userInput=''"
          />
          <WebView
            v-else
            :visible="i===activePageIndex"
            :key="'webview'+page.id"
            style="position:absolute;left:0;top:0;right:0;bottom:0;"
            :href.sync="page.href"
            :nav="page.nav"
            :zoomfactor.sync="page.zoomFactor"
            @update:status="page.updateStatus($event)"
            @update:href="page.userInput=''"
            @update:wheel="onWebviewWheel"
          />
        </template>
      </Swiper>
    </v-content>
  </v-app>
</template>
<script lang="ts">
import { Component, Vue, Watch } from "vue-property-decorator";
import { remote } from "electron";
import Vendor from "./vendor";
import Launcher from "./launcher";
import * as UrlUtils from "@/common/url-utils";
import { State } from "vuex-class";

class Page {
  static nextId = 0;

  id = Page.nextId++;
  userInput = "";
  zoomFactor = 0;

  _href = "";
  _savedWebviewHref = "";
  private readonly status: WebView.Status = {
    title: "",
    favicon: "",
    progress: 0,
    canGoBack: false,
    canGoForward: false,
    cert: null
  };

  readonly nav: WebView.Nav = {
    goBack: 0,
    goForward: 0,
    reload: 0,
    reloadIgnoringCache: 0,
    stop: 0,
    reGo: 0
  };

  constructor(href: string) {
    this.href = href;
  }
  get href() {
    return this._href;
  }
  set href(val: string) {
    if (val !== this._href) {
      this._savedWebviewHref = "";
      this._href = val;
    }
  }

  updateStatus(status: WebView.Status) {
    Object.assign(this.status, status);
  }
  get title() {
    return this.status.title || this.href;
  }
  get favicon() {
    return this.status.favicon;
  }
  get progress() {
    return this.isBuiltin ? 100 : this.status.progress * 100;
  }
  get loading() {
    return this.status.progress !== 1;
  }
  get isBuiltin() {
    return !this.href || this.href.toLowerCase().startsWith("sync:");
  }
  get canGoBack() {
    if (!this.isBuiltin && !this.status.canGoBack) {
      return true;
    }
    return this.status.canGoBack;
  }
  get canGoForward() {
    if (this.isBuiltin && !this.status.canGoForward && this._savedWebviewHref) {
      return true;
    }
    return this.status.canGoForward;
  }
  get cert() {
    return this.status.cert;
  }

  goBack() {
    if (!this.isBuiltin && !this.status.canGoBack) {
      const href = this.href;
      this.href = "";
      this._savedWebviewHref = href;
    } else {
      this.nav.goBack++;
    }
  }
  goForward() {
    if (this.isBuiltin && !this.status.canGoForward && this._savedWebviewHref) {
      this.href = this._savedWebviewHref;
    } else {
      this.nav.goForward++;
    }
  }
  reloadOrStop(shiftKey: boolean) {
    if (this.progress === 100) {
      if (shiftKey) {
        this.nav.reloadIgnoringCache++;
      } else {
        this.nav.reload++;
      }
    } else {
      this.nav.stop++;
    }
    this.userInput = "";
  }
  reload() {
    this.nav.reload++;
    this.userInput = "";
  }
  forceReload() {
    this.nav.reloadIgnoringCache++;
    this.userInput = "";
  }
  zoomIn() {
    this.zoomFactor = Math.min(3, this.zoomFactor + 0.2);
  }
  zoomOut() {
    this.zoomFactor = Math.max(0.5, this.zoomFactor - 0.1);
  }
}

type OpenTab = {
  href: string;
  title?: string;
  mode?: "append-active" | "inplace";
};

@Component({
  components: {
    Vendor,
    Launcher
  }
})
export default class Nova extends Vue {
  pages: Page[] = [];
  activePageIndex = 0;
  get activePage() {
    return this.pages[this.activePageIndex];
  }
  urlBoxFocused = false;

  accessHistoryPosition = { x: 0, y: 0, width: 0 };
  showAccessHistory = false;
  keyword = "";
  isDarwin = process.platform === "darwin";

  @State shortcuts!: entities.Shortcut[];

  @Watch("activePage")
  activePageChanged() {
    (this.$refs.urlBoxWithIcon as Element).querySelector("input")!.blur();
  }

  get showAddShortcutBtn() {
    return !this.activePage.isBuiltin && !this.activePage.userInput;
  }

  get shortcutAdded() {
    return (
      (this.shortcuts || []).findIndex(s => s.href === this.activePage.href) >=
      0
    );
  }

  closeTab(index: number) {
    if (this.pages.length < 2) {
      remote.getCurrentWindow().close();
      return;
    }

    this.pages.splice(index, 1);
    if (index < this.activePageIndex) {
      this.activePageIndex--;
    } else {
      this.activePageIndex = Math.min(
        this.pages.length - 1,
        this.activePageIndex
      );
    }
  }

  openTab(
    href: string,
    mode?: "append" | "append-active" | "inplace" | "inplace-builtin"
  ) {
    const formalized = UrlUtils.formalize(href);
    if (mode === "append-active") {
      const page = new Page(formalized);
      this.pages.splice(this.activePageIndex + 1, 0, page);
      this.activePageIndex++;
    } else if (mode === "inplace") {
      this.activePage.href = formalized;
    } else if (mode === "inplace-builtin") {
      if (this.activePage.isBuiltin) {
        this.activePage.href = formalized;
      } else {
        this.openTab(href, "append");
      }
    } else {
      const page = new Page(formalized);
      this.pages.push(page);
      this.activePageIndex = this.pages.length - 1;
    }
  }

  navigateTo(href: string) {
    const newHref = UrlUtils.formalize(href);
    if (newHref === this.activePage.href) {
      this.activePage.nav.reGo++;
    } else {
      this.activePage.href = newHref;
    }
  }
  beforeMount() {
    (document.querySelector(".splash")! as HTMLElement).style.display = "none";
  }
  async created() {
    const mq = remote.app.EXTENSION.mq;
    const tabActionTopic = `TabAction-${remote.getCurrentWindow().id}`;
    const initTabAction = mq.peek(tabActionTopic) as TabAction | null;
    if (initTabAction && initTabAction.action === "new" && initTabAction.url) {
      this.openTab(initTabAction.url);
    } else {
      this.pages.push(new Page(""));
    }
    BUS.$on("open-tab", (data: OpenTab) => {
      this.openTab(data.href, data.mode);
    });

    for (;;) {
      const action = (await mq.read(
        tabActionTopic,
        remote.getCurrentWebContents().id
      )) as TabAction;
      if (this.isModaling()) {
        continue;
      }
      if (action.action === "close") {
        this.closeTab(this.activePageIndex);
      } else if (action.action === "new") {
        this.openTab(action.url || "");
      }
    }
  }

  onDblClickTitleBar() {
    const action = (() => {
      if (process.platform === "darwin") {
        return remote.systemPreferences.getUserDefault(
          "AppleActionOnDoubleClick",
          "string"
        ) as string;
      }
    })();
    const win = remote.getCurrentWindow();
    let payload: WindowAction | undefined;
    switch (action) {
      case "Minimize":
        payload = {
          windowId: win.id,
          action: "minimize"
        };
        break;
      case "None":
        break;
      default:
        if (win.isMaximized()) {
          payload = {
            windowId: win.id,
            action: "unmaximize"
          };
        } else {
          payload = {
            windowId: win.id,
            action: "maximize"
          };
        }
    }
    if (payload) {
      remote.app.EXTENSION.mq.post("WindowAction", payload);
    }
  }

  isModaling() {
    return !!this.$el.querySelector(".v-overlay--active");
  }

  updateAccessHistoryLayout() {
    const rect = (this.$refs.urlBoxWithIcon as Element)
      .getClientRects()
      .item(0)!;
    this.accessHistoryPosition.x = Math.round(rect.left);
    this.accessHistoryPosition.y = Math.round(rect.bottom) + 5;

    const navBoxRect = (this.$refs.navBox as Element).getClientRects().item(0)!;
    this.accessHistoryPosition.width = Math.round(navBoxRect.right - rect.left);
  }

  onResize() {
    this.updateAccessHistoryLayout();
  }

  onUrlBoxInput(val: string) {
    val = val.trim();
    if (val.length > 0) {
      this.updateAccessHistoryLayout();
      this.keyword = val;
      this.showAccessHistory = true;
    } else {
      this.showAccessHistory = false;
    }
  }
  onUrlBoxBlur() {
    this.urlBoxFocused = false;
    this.showAccessHistory = false;
  }
  onAccessHistorySelected(val: AccessHistoryPanel.Item) {
    this.activePage.href = val.href;
    this.activePage.userInput = "";
  }

  onSwipe(dir: string) {
    if (dir === "right") {
      this.activePage.goBack();
    } else {
      this.activePage.goForward();
    }
  }

  addOrRemoveShortcut() {
    if (this.shortcutAdded) {
      GDB.shortcuts.bulkDelete(
        this.shortcuts
          .filter(s => s.href === this.activePage.href)
          .map(s => s.id!)
      );
    } else {
      GDB.shortcuts.add({
        title: this.activePage.title,
        href: this.activePage.href
      });
    }
  }
  onWebviewWheel(delta: { x: number; y: number }) {
    (this.$refs.swiper as any).handleWheel(delta.x, delta.y);
  }

  get menuItems(): WindowMenu.Item[] {
    const isDarwin = process.platform === "darwin";
    return [
      {
        label: "New Tab",
        keys: isDarwin ? ["command+t"] : ["ctrl+t"],
        action: () => this.openTab("")
      },
      {
        label: "Close Tab",
        keys: isDarwin ? ["command+w"] : ["ctrl+w"],
        action: () => this.closeTab(this.activePageIndex)
      },
      {
        label: "Reload",
        keys: isDarwin ? ["command+r"] : ["f5", "ctrl+r"],
        disabled: this.activePage.isBuiltin,
        action: () => {
          this.activePage.reload();
        }
      },
      {
        label: "Force Reload",
        keys: isDarwin ? ["command+shift+r"] : ["ctrl+shift+r"],
        disabled: this.activePage.isBuiltin,
        invisible: true,
        action: () => {
          this.activePage.forceReload();
        }
      },
      {
        label: "Go Back",
        keys: isDarwin ? ["command+["] : ["ctrl+["],
        disabled: !this.activePage.canGoBack,
        action: () => {
          this.activePage.goBack();
        }
      },
      {
        label: "Go Forward",
        keys: isDarwin ? ["command+]"] : ["ctrl+]"],
        disabled: !this.activePage.canGoForward,
        action: () => {
          this.activePage.goForward();
        }
      },
      {
        label: "Zoom In",
        keys: isDarwin ? ["command+="] : ["ctrl+="],
        disabled: this.activePage.isBuiltin,
        action: () => {
          this.activePage.zoomIn();
        },
        divider: true
      },
      {
        label: "Zoom Out",
        keys: isDarwin ? ["command+-"] : ["ctrl+-"],
        disabled: this.activePage.isBuiltin,
        action: () => {
          this.activePage.zoomOut();
        }
      },
      {
        label: "Reset Zoom",
        keys: isDarwin ? ["command+0"] : ["ctrl+0"],
        disabled: this.activePage.isBuiltin,
        action: () => {
          this.activePage.zoomFactor = 1;
        }
      },
      {
        label: "Wallets",
        keys: [],
        action: () => {
          this.openTab("sync://wallets", "inplace-builtin");
        },
        divider: true
      },
      {
        label: "Settings",
        keys: isDarwin ? ["command+,"] : [],
        action: () => {
          this.openTab("sync://settings", "inplace-builtin");
        }
      },
      {
        label: "Toggle Developer Tools",
        keys: [],
        action: () => {
          remote.getCurrentWebContents().toggleDevTools();
        },
        divider: true
      },
      {
        label: "About",
        keys: [],
        action: () => {
          remote.app.EXTENSION.showAbout();
        }
      }
    ];
  }

  get experimentalItems(): WindowMenu.Item[] {
    const isDarwin = process.platform === "darwin";
    return [
      {
        label: "Bail Out",
        keys: [],
        action: () => {
          this.openTab("sync://slashing/bailout", "inplace-builtin");
        },
        divider: true
      },
      {
        label: "Locked Transfer",
        keys: [],
        action: () => {
          this.openTab("sync://locked/transfer", "inplace-builtin");
        }
      }
    ];
  }
}
</script>

<style lang="scss">
html {
  overflow: hidden; // vuetify will set this value to 'scroll', overwrite it
}

.toolbar {
  overflow: hidden;
}
.html-full-screen .toolbar {
  display: none;
}

.theme--light .toolbar {
  background-color: #e6e6e6;
}

.theme--dark .toolbar {
  background-color: #404040;
}

.blur .theme--light .toolbar {
  background-color: #f4f4f4;
}
.blur .theme--dark .toolbar {
  background-color: #383838;
}

.tab-bar {
  overflow: hidden;
  padding: 8px 40px 0px 8px;
  transition: padding 0.2s;
}

.darwin .tab-bar {
  padding-left: 80px;
}

.darwin.full-screen .tab-bar {
  padding-left: 8px;
  padding-top: 3px;
}
.tab-button {
  flex: 0 1 auto;
  width: 220px;
}
.drag {
  -webkit-app-region: drag;
}
.no-drag {
  -webkit-app-region: no-drag;
}

.tab-tools {
  float: right;
}

.label {
  color: #fff;
  padding: 0px 4px;
  border-radius: 2px;
  font-size: 10px;
}
.break-all {
  word-break: break-all;
}
.theme--light.v-menu__content {
  box-shadow: 0px 5px 24px 2px rgba(0, 0, 0, 0.25),
    rgba(0, 0, 0, 0.5) 0px 0px 0.5px 0px;
  border-radius: 6px;
}
.theme--light .v-dialog {
  box-shadow: 0px 11px 15px -7px rgba(0, 0, 0, 0.2),
    0px 24px 38px 3px rgba(0, 0, 0, 0.14), 0px 9px 46px 8px rgba(0, 0, 0, 0.12),
    rgba(0, 0, 0, 0.2) 0px 0px 0px 0.5px;
  border-radius: 6px;
}

.theme--light .outline {
  position: relative;
}
.theme--light .outline::before {
  pointer-events: none;
  border-radius: inherit;
  content: "";
  position: absolute;
  left: 0;
  top: 0;
  height: 100%;
  width: 100%;
  box-shadow: rgba(0, 0, 0, 0.25) 0px 0px 0px 0.5px inset;
}
.v-overlay--active:before {
  opacity: 0.1;
}
.v-expansion-panel__header {
  padding-right: 12px !important;
}
.v-expansion-panel__header__icon {
  margin-left: 12px !important;
}

.tab-button-enter {
  width: 0px;
  opacity: 0;
}
.tab-button-leave-active {
  display: none;
}

.nav-bar {
  box-shadow: 0px 2px 3px 1px rgba(0, 0, 0, 0.15);
}
.theme--light .nav-bar {
  background-color: #ffffff;
}
.theme--dark .nav-bar {
  background-color: #212121;
}

.nav-box {
  background-color: rgba(0, 0, 0, 0.05);
  border-radius: 4px;
  overflow: hidden;
  height: 24px;
  box-shadow: 0px 0px 0px 0.5px rgba(0, 0, 0, 0.05) inset;
  flex: 1 1 auto;
}

.nav-box-focused {
  box-shadow: 0px 0px 0px 2px rgba(25, 118, 210, 0.7);
}

.url-box {
  width: 100%;
  height: 100%;
  outline: none;
  color: rgba(0, 0, 0, 0.5);
  cursor: default;
  font-size: 13px;
}
.url-box:focus {
  color: rgba(0, 0, 0, 0.8);
  cursor: text;
}
.url-box::placeholder {
  color: rgba(0, 0, 0, 0.35);
}

.url-box-with-icon:focus-within {
  background-color: #ffffff;
  box-shadow: 0px 0px 0px 1px rgba(0, 0, 0, 0.1);
}
.theme--light.v-btn .v-btn__content .v-icon {
  color: rgba(0, 0, 0, 0.6);
}
.sharp-line {
  height: 1px;
  background-image: linear-gradient(
    to top,
    black 0%,
    black 51%,
    transparent 51%
  );
  background-size: 100% 1px;
  opacity: 0.15;
}
.theme--light.application {
  color: rgba(0, 0, 0, 0.8);
}
.sign-dialog {
  position: fixed;
  right: 0;
  bottom: 0;
  width: auto;
}
.sign-dialog-transition-enter,
.sign-dialog-transition-leave-to {
  transform: translateY(calc(100% + 40px));
}
.serif {
  font-family: "Roboto Slab";
}

@keyframes app-fadein {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.app-fadein {
  animation: 0.6s app-fadein;
}
</style>
