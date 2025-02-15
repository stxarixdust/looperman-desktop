<template>
  <div
    :class="theme + ' ' + (compactMode ? 'sizing-compact' : 'sizing-regular')"
    class="appRoot"
  >
    <Transition name="popup">
      <Settings
        ref="Settings"
        v-if="settingsOpen"
        @close="
          settingsOpen = false;
          reloadPrefs();
        "
        :kdeThemeName="kdePrefs.themeName"
        :hasKdeTheme="kdePrefs.exists"
      />
    </Transition>
    <TitleBar ref="TitleBar" @settingsOpen="settingsOpen = !settingsOpen" />
    <MainContent ref="MainContent" />
    <PlayBar ref="PlayBar" />
  </div>
</template>

<script>
import TitleBar from "./components/TitleBar.vue";
import MainContent from "./components/MainContent.vue";
import PlayBar from "./components/PlayBar.vue";
import Settings from "./components/Settings.vue";
import "@fontsource/rubik/400.css";
import "@fontsource/rubik/600.css";
import "@fontsource/rubik/700.css";
import "@ibm/plex/scss/ibm-plex.scss";

const electron = window.require("electron");

export default {
  name: "App",
  components: {
    TitleBar,
    MainContent,
    PlayBar,
    Settings,
  },
  methods: {
    search() {
      let originalQuery = this.$refs.TitleBar.$refs.SearchBar.value;
      let topBarQuery = JSON.parse(
        JSON.stringify(this.$refs.MainContent.$refs.TopArea.queryInfo)
      );
      let query = {
        category: topBarQuery.category,
        keys: "",
        order: topBarQuery.order,
        tempo: topBarQuery.tempo,
        page: 1,
        key: topBarQuery.key,
        date: topBarQuery.date,
        genre: topBarQuery.genre,
        filterByKey: topBarQuery.key[0] != "",
        author: topBarQuery.author,
      };

      // This matches every ocurrence of (NUMBER) BPM in the query
      let bpmregex = /[0-9]+ *bpm(?: |$)/gi;
      let bpmtext = Array.from(originalQuery.matchAll(bpmregex));
      query.keys = originalQuery.replaceAll(bpmregex, "");

      if (bpmtext.length > 0) {
        query.tempo = bpmtext.pop()[0].match(/[1-9][0-9]*/)[0];
        query.tempo = [Number(query.tempo), Number(query.tempo)];
        this.$refs.MainContent.$refs.TopArea.setTempo(query.tempo);
      }

      // This matches every occurence of a key in the query
      let keyregex = /([a-g]|A-G) *[#b]? *(maj|min)(or)?(?: |$)/gi;
      let keytext = Array.from(originalQuery.matchAll(keyregex));
      query.keys = query.keys.replaceAll(keyregex, "");

      // Remove BPM/Key information from query
      this.$refs.TitleBar.$refs.SearchBar.value = query.keys
        .replaceAll("  ", " ")
        .trim();

      if (keytext.length > 0) {
        query.filterByKey = true;
        let keymatch = keytext.pop()[0].toLowerCase();
        let note = keymatch[0];
        if (keymatch[1] == "#") note += "s";
        if (keymatch[1] == "b") {
          let charcode = keymatch.charCodeAt(1) - "a".charCodeAt(0) + 1;
          charcode %= 8;
          charcode += "a".charCodeAt(0);
          note = String.fromCharCode(charcode);
        }
        query.key = [
          note,
          keymatch.match(/(maj|min)/i)[0].toLowerCase() == "min" ? "m" : "",
        ];
      }

      console.log(query);
      this.$refs.MainContent.$refs.TopArea.$data.query = JSON.parse(
        JSON.stringify(query)
      );
      this.$refs.MainContent.$refs.Results.$data.query = JSON.parse(
        JSON.stringify(query)
      );
      this.$refs.MainContent.$refs.Results.reset();
    },
    async reloadPrefs() {
      let prefs = await electron.ipcRenderer.invoke("getPrefs");

      this.theme = prefs.theme;
      this.compactMode = prefs.compactMode;

      if (prefs.keys.germanic) {
        console.log("GERMANIC");
        this.keyLabels = {
          sharps: ["C", "D", "X", "F", "G", "A"],
          keys: ["C", "D", "E", "F", "G", "A", "H"],
        };
        console.log(this.keyLabels);
      } else {
        console.log("NOT GERMANIC");
        this.keyLabels = {
          sharps: ["C", "D", "X", "F", "G", "A"],
          keys: ["C", "D", "E", "F", "G", "A", "B"],
        };
      }
    },
  },
  data() {
    return {
      keyLabels: {
        sharps: ["C", "D", "X", "F", "G", "A"],
        keys: ["C", "D", "E", "F", "G", "A", "B"],
      },
      settingsOpen: false,
      theme: "theme-dark",
      compactMode: false,
      kdePrefs: {
        exists: false,
        themeName: "System theme",
        colors: {
          accent: ["inherit", "inherit", "inherit", "inherit"],
          background: ["inherit", "inherit", "inherit", "inherit"],
          foreground: ["inherit", "inherit", "inherit", "inherit"],
        },
        font: ["inherit", 0],
        monoFont: ["inherit", 0],
      },
    };
  },
  async beforeCreate() {
    let rawPrefs = await electron.ipcRenderer.invoke("getKDEPrefs");
    this.kdePrefs = {
      exists: true,
      themeName: rawPrefs.General.ColorScheme,
      colors: {
        accent: [
          `rgb(${rawPrefs["Colors:Selection"].BackgroundNormal})`,
          `rgb(${rawPrefs["Colors:Selection"].BackgroundNormal})`,
          `rgb(${rawPrefs["Colors:Selection"].BackgroundNormal})`,
          `rgb(${rawPrefs["Colors:Selection"].BackgroundNormal})`,
        ],
        background: [
          `rgb(${rawPrefs.WM.activeBackground})`,
          `rgb(${rawPrefs["Colors:Window"].BackgroundNormal})`,
          `rgb(${rawPrefs["Colors:Button"].BackgroundNormal})`,
          `rgb(${rawPrefs["Colors:View"].BackgroundNormal})`,
        ],
        foreground: [
          `rgb(${rawPrefs["Colors:Window"].ForegroundNormal})`,
          `rgb(${rawPrefs["Colors:Window"].ForegroundNormal})`,
          `rgb(${rawPrefs["Colors:Window"].ForegroundNormal})`,
          `rgb(${rawPrefs["Colors:Window"].ForegroundNormal})`,
        ],
      },
      font: [
        `"${rawPrefs.General.font.split(",")[0]}"`,
        rawPrefs.General.font.split(",")[1],
      ],
      monoFont: [
        `"${rawPrefs.General.fixed.split(",")[0]}"`,
        rawPrefs.General.fixed.split(",")[1],
      ],
    };
    this.reloadPrefs();
  },
};
</script>

<style>
.theme-kde-native {
  --accent-1-100: v-bind("kdePrefs.colors.accent[0]");
  --accent-2-100: v-bind("kdePrefs.colors.accent[0]");
  --accent-3-100: v-bind("kdePrefs.colors.accent[0]");
  --accent-4-100: v-bind("kdePrefs.colors.accent[0]");

  --accent-1-200: v-bind("kdePrefs.colors.accent[1]");
  --accent-2-200: v-bind("kdePrefs.colors.accent[1]");
  --accent-3-200: v-bind("kdePrefs.colors.accent[1]");
  --accent-4-200: v-bind("kdePrefs.colors.accent[1]");

  --accent-1-300: v-bind("kdePrefs.colors.accent[2]");
  --accent-2-300: v-bind("kdePrefs.colors.accent[2]");
  --accent-3-300: v-bind("kdePrefs.colors.accent[2]");
  --accent-4-300: v-bind("kdePrefs.colors.accent[2]");

  --background-100: v-bind("kdePrefs.colors.background[0]");
  --background-200: v-bind("kdePrefs.colors.background[1]");
  --background-300: v-bind("kdePrefs.colors.background[2]");
  --background-400: v-bind("kdePrefs.colors.background[3]");

  --soft-glass-100: v-bind("kdePrefs.colors.background[0]");
  --soft-glass-200: v-bind("kdePrefs.colors.background[1]");

  --glass-100: v-bind("kdePrefs.colors.background[0]");
  --glass-200: v-bind("kdePrefs.colors.background[1]");

  --foreground-100: v-bind("kdePrefs.colors.foreground[0]");
  --foreground-200: v-bind("kdePrefs.colors.foreground[1]");
  --foreground-300: v-bind("kdePrefs.colors.foreground[2]");
  --foreground-400: v-bind("kdePrefs.colors.foreground[3]");

  --inverse-foreground: v-bind("kdePrefs.colors.background[0]");

  /*
  --glass-100: #{rgba(darken($background, 16%), 0.5)};
  --glass-200: #{rgba(darken($background, 12%), 0.5)};

  --soft-glass-100: #{rgba(darken($background, 16%), 0.85)};
  --soft-glass-200: #{rgba(darken($background, 12%), 0.85)};

  // Scroll bars

  --scroll-thumb-color: #{rgba($foreground, 25%)};
  --scroll-thumb-color-hover: #{rgba($foreground, 50%)}; */
  font-family: v-bind("kdePrefs.font[0]") !important;
  --kde-font-size: v-bind("kdePrefs.font[1] + 'px'");
  --font-size: calc(var(--kde-font-size) * var(--font_scale) * 1.35) !important;
}
.theme-kde-native .results {
  background-color: v-bind("kdePrefs.colors.background[3]");
}
.theme-kde-native .mono-value {
  font-family: v-bind("kdePrefs.monoFont[0]") !important;
  font-size: v-bind("kdePrefs.monoFont[1]") !important;
}
</style>

<style lang="scss">
@import "./styles/sizing-regular.scss";
@import "./styles/sizing-compact.scss";
@import "./styles/globals.scss";

@import "./styles/themes/chroma.scss";
@import "./styles/themes/dark.scss";
@import "./styles/themes/dracula.scss";
@import "./styles/themes/light.scss";
@import "./styles/themes/purple.scss";
@import "./styles/themes/dino.scss";

@import "./styles/components/MainContent.scss";
@import "./styles/components/PlayBar.scss";
@import "./styles/components/Results.scss";
@import "./styles/components/Settings.scss";
@import "./styles/components/TitleBar.scss";
@import "./styles/components/TopArea.scss";

@import "./styles/inputs/Button.scss";
@import "./styles/inputs/FileSelector.scss";
@import "./styles/inputs/SearchBar.scss";
@import "./styles/inputs/Select.scss";
@import "./styles/inputs/Slider.scss";
@import "./styles/inputs/Switch.scss";
@import "./styles/inputs/Tag.scss";

.popup-enter-active,
.popup-leave-active {
  transition: all var(--animation-duration-alt) var(--animation-timing-alt);
}

.popup-enter-from,
.popup-leave-to {
  transform: translateY(calc(64px)) scale(0.97);
  opacity: 0;
  box-shadow: 0px 0px 0px 0px #000;
}

.popup-enter-to,
.popup-leave-from {
  transform: none;
  opacity: 1;
}

body {
  margin: 0;
  overflow-y: overlay;
}

.appRoot {
  color: var(--foreground-100);
  font-size: var(--font-size);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  background: var(--background-200);
  .title-bar {
    top: 0;
  }
  .title-bar,
  .play-bar {
    z-index: 5;
    position: fixed;
    box-sizing: border-box;
    width: 100%;
  }
  .main-content {
    z-index: -1;
  }
  .play-bar {
    bottom: 0;
  }
}
*::-webkit-scrollbar {
  width: 10px;
  margin-right: 10px;
}

*::-webkit-scrollbar-track {
  background: transparent;
}

*::-webkit-scrollbar-thumb {
  transition: var(--animation-duration);
  background: var(--scroll-thumb-color);
  &:hover {
    background: var(--scroll-thumb-color-hover);
  }
  border-radius: 10px;
}
@media only screen and (max-width: 1200px) {
  h1 {
    font-size: calc(
      var(--font-size) * var(--heading-scale) * var(--heading-scale) *
        var(--heading-scale) * var(--heading-scale) * var(--scale)
    );
  }
  .play-bar {
    .player-controls .player-buttons,
    .side-playbar-controls {
      gap: var(--item-gap);
    }
  }
}
@media only screen and (max-width: 800px) {
  .mono-value {
    letter-spacing: -0.5px;
  }
  .info-duration {
    display: none;
  }
  .result {
    grid-template-columns: var(--profile-pic-large-size) auto 6ch 5ch;
    gap: calc(var(--item-gap) / 1.5);
    row-gap: calc(var(--item-gap) / 2);
    grid-template-areas:
      "p d k b "
      "v v a a ";
    .result-profile-pic {
      grid-area: p;
    }
    .visualizer {
      grid-area: v;
    }
    .actions-result {
      grid-area: a;
    }
    .info-key {
      grid-area: k;
    }
    .info-bpm {
      grid-area: b;
    }
    .info-audio {
      grid-area: d;
    }
  }
  .player-controls {
    flex-grow: 2;
    flex-shrink: 2;
  }
  #volume-btt-spacer,
  #volumeSlider-btt {
    display: block;
  }
  .play-bar .default-slider {
    display: none;
  }
  .play-bar .profile-picture {
    width: calc(var(--profile-pic-large-size) * 0.8);
  }
  .results-header {
    grid-template-columns: var(--profile-pic-large-size) auto 6ch 5ch;
    gap: calc(var(--item-gap) / 1.5);
  }
  .settings-window {
    flex-direction: column;
    .content {
      @include glass(true, true);
      background: var(--soft-glass-200);
      border-top-left-radius: 0;
      border-top-right-radius: 0;
      border-top: 1px solid var(--seperator);
    }
    .sidebar {
      @include glass(false, true);
      background: var(--glass-200);
      border-bottom-left-radius: 0;
      border-bottom-right-radius: 0;
      border-bottom: none;
      .last {
        position: relative;
      }
    }
    #sidebar-x {
      display: block;
    }
    #content-x {
      display: none;
    }
  }
}
@media only screen and (max-width: 600px) {
  .settings-window .content .inner-content .input-component {
    margin-left: 0px;
    width: 100%;
  }
  .play-bar .audio-desc {
    display: none;
  }
  .play-bar .info-audio {
    flex-grow: 0;
    -webkit-mask-image: none;
  }
  .settings-window {
    margin-left: -50vw;
    width: 100vw;
    height: calc(100vh - (2 * var(--side-padding)));
    margin-top: calc(-0.5 * (100vh - (2 * var(--side-padding))));
  }
}
@media only screen and (max-width: 500px) {
  .result {
    grid-template-columns: var(--profile-pic-large-size) auto 6ch;
    grid-template-rows: auto;
    grid-template-areas:
      "p d k"
      "p d b"
      "v v v"
      "a a a";
    .actions-result {
      width: 100%;
    }
  }
  .results-header {
    grid-template-columns: var(--profile-pic-large-size) auto auto;
    gap: calc(var(--item-gap) / 1.5);
    div:nth-of-type(3) {
      text-align: right;
      &::after {
        content: " & Tempo";
      }
    }
    div:nth-of-type(4) {
      display: none;
    }
  }
  .hide-bp-500 {
    display: none;
  }
  .play-bar .player-controls .player-buttons,
  .play-bar .side-playbar-controls {
    flex-grow: 0;
  }
  h1 {
    font-size: calc(
      var(--font-size) * var(--heading-scale) * var(--heading-scale) *
        var(--heading-scale) * var(--scale)
    );
  }
}
@media only screen and (max-width: 400px) {
  #topSearch {
    display: none;
  }
  #bottomSearch {
    display: block;
  }
  h1 {
    font-size: calc(
      var(--font-size) * var(--heading-scale) * var(--heading-scale) *
        var(--scale)
    );
  }
  .hide-bp-400 {
    display: none;
  }
  .settings-window {
    .content,
    .sidebar a,
    .sidebar h2,
    #version-string {
      padding-left: var(--item-gap);
      padding-right: var(--item-gap);
    }
    .sidebar,
    .content {
      padding-top: var(--item-gap);
    }
  }
}
@media only screen and (max-width: 300px) {
  .result {
    grid-template-columns: var(--profile-pic-large-size) auto;
    grid-template-areas:
      "p k"
      "p b"
      "d d"
      "v v"
      "a a";
  }
  .hide-bp-300 {
    display: none;
  }
  h1 {
    font-size: calc(var(--font-size) * var(--scale) * var(--heading-scale));
  }
  .settings-window .vue-feather,
  .settings-window .content .inner-content .section .ico-heading {
    display: none;
  }
}
</style>
