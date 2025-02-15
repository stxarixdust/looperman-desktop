<template>
  <div class="top-area">
    <p ref="LibraryName" class="LibraryName">
      {{
        query.keys == undefined || query.keys == ""
          ? library_name
          : "Search results for:"
      }}
    </p>
    <h1>
      <span v-if="query.keys == undefined || query.keys == ''">{{
        query.keys
      }}</span>
      {{ section_name }}
      <span v-if="!(query.author == undefined || query.author == '')"
        >by {{ query.author }}</span
      >
    </h1>
    <SearchBar
      placeholder="Search for loops..."
      @submitSearch="this.$parent.$parent.search()"
      ref="SearchBar"
      id="bottomSearch"
    />
    <div class="tags-container" v-if="search_header">
      <div class="tags-left">
        <Tag
          title="Tempo"
          :additionalInfo="
            (queryInfo.tempo[0] == queryInfo.tempo[1]
              ? queryInfo.tempo[0]
              : queryInfo.tempo[0] + ' - ' + queryInfo.tempo[1]) + 'BPM'
          "
          icon="activity"
          @cancel="filterValues.tempoRange = queryInfo.tempo"
          @save="
            queryInfo.tempo = filterValues.tempoRange;
            this.$parent.$parent.search();
          "
          @remove="
            queryInfo.tempo = [0, 200];
            this.$parent.$parent.search();
          "
          :saveCondition="
            filterValues.tempoRange.toString() != [0, 200].toString()
          "
        >
          <vue-slider
            v-model="filterValues.tempoRange"
            id="playbackSlider"
            ref="playbackSlider"
            :lazy="true"
            :min="0"
            :max="200"
            :interval="1"
          />
        </Tag>
        <Tag
          title="Key"
          :additionalInfo="
            (queryInfo.key[0].charAt(0).toUpperCase() == 'B' &&
            this.$parent.$parent.keyLabels.keys[6] == 'H'
              ? 'H'
              : 'B') +
            (queryInfo.key[0].charAt(1) == 's' ? '#' : '') +
            ' ' +
            (queryInfo.key[1] == 'm' ? 'Minor' : 'Major')
          "
          icon="music"
          @cancel="filterValues.key = [...queryInfo.key]"
          @save="
            queryInfo.key = [...filterValues.key];
            this.$parent.$parent.search();
          "
          @remove="
            queryInfo.key = ['', ''];
            filterValues.key = ['', ''];
            this.$parent.$parent.search();
          "
          :saveCondition="filterValues.key[0] != ''"
        >
          <Switch
            class="keyModeSwitch"
            :modelValue="filterValues.key[1] == 'm'"
            @update:modelValue="(v) => (filterValues.key[1] = v ? 'm' : '')"
            :title="'Mode: ' + (filterValues.key[1] == 'm' ? 'Minor' : 'Major')"
          />
          <div>
            <div class="key-row sharps">
              <div
                v-for="(label, i) in this.$parent.$parent.keyLabels.sharps"
                :key="i"
                @click="filterValues.key[0] = keyData.sharps[i]"
                :class="{
                  selected: filterValues.key[0] == keyData.sharps[i],
                  'key-seperator': label == 'X',
                }"
              >
                {{ label != "X" ? label + "#" : "" }}
              </div>
            </div>
            <div class="key-row">
              <div
                v-for="(label, i) in this.$parent.$parent.keyLabels.keys"
                :key="i"
                @click="filterValues.key[0] = keyData.keys[i]"
                :class="{
                  selected: filterValues.key[0] == keyData.keys[i],
                  'key-seperator': label == 'X',
                }"
              >
                {{ label != "X" ? label : "" }}
              </div>
            </div>
          </div>
        </Tag>
        <div
          class="tag active"
          v-if="!(query.author == undefined || query.author == '')"
        >
          <vue-feather type="user" size="18" />
          <div>
            User<span v-if="!(query.author == undefined || query.author == '')"
              >: {{ query.author }}</span
            >
          </div>
          <vue-feather
            type="x"
            size="18"
            @click="
              queryInfo.author = '';
              this.$parent.$parent.search();
            "
            v-if="!(query.author == undefined || query.author == '')"
          />
        </div>
        <div>
          <div
            class="tag"
            @click.self="filterEnabled.genre = !filterEnabled.genre"
            :class="{ active: queryInfo.genre != 0 }"
          >
            <div
              @click="
                filterEnabled.genre = !filterEnabled.genre;
                filterValues.genre = queryInfo.genre;
              "
            >
              Genre<span v-if="queryInfo.genre != 0" class="hide-bp-300"
                >: {{ genrelist.find((e) => e[0] == queryInfo.genre)[1] }}</span
              >
            </div>
            <vue-feather
              type="x"
              size="18"
              @click="
                queryInfo.genre = 0;
                this.$parent.$parent.search();
              "
              v-if="queryInfo.genre != 0"
            />
          </div>
          <Transition name="tag-popup">
            <div class="tag-popout" v-if="filterEnabled.genre">
              <h2>Genre</h2>
              <div class="select">
                <div
                  class="option"
                  v-for="genre in genrelist"
                  :key="genre[0]"
                  :class="{ selected: parseInt(genre[0]) == queryInfo.genre }"
                  @click="
                    queryInfo.genre = parseInt(genre[0]);
                    this.$parent.$parent.search();
                    filterEnabled.genre = false;
                    filterAdded.genre = true;
                  "
                >
                  {{ genre[1] }}
                </div>
              </div>
            </div>
          </Transition>
        </div>
      </div>
      <div class="tag-duo">
        <div @click="filterEnabled.sort = !filterEnabled.sort">
          Sort by {{ queryInfo.order[0] }}
        </div>
        <vue-feather
          :type="queryInfo.order[1] == 'd' ? 'chevron-down' : 'chevron-up'"
          size="18"
          @click="
            queryInfo.order[1] = queryInfo.order[1] == 'd' ? 'a' : 'd';
            this.$parent.$parent.search();
          "
        />
      </div>
      <Transition name="tag-popup-l">
        <div class="tag-popout align-left" v-if="filterEnabled.sort">
          <h2>Sort by</h2>
          <div class="select">
            <div
              class="option"
              v-for="sort in [
                ['date', 'Date'],
                ['name', 'Title'],
                ['author', 'Author'],
                ['tempo', 'Tempo'],
                ['downloads', 'Downloads'],
                ['comments', 'Comments'],
              ]"
              :key="sort[0]"
              :class="{ selected: sort[0] == queryInfo.order[0] }"
              @click="
                queryInfo.order[0] = sort[0];
                this.$parent.$parent.search();
                filterEnabled.sort = false;
              "
            >
              {{ sort[1] }}
            </div>
          </div>
        </div>
      </Transition>
    </div>
    <div class="results-header" v-if="search_header">
      <div>User</div>
      <div>Title</div>
      <div>Key</div>
      <div>Tempo</div>
    </div>
  </div>
</template>

<script>
import VueFeather from "vue-feather";
import VueSlider from "vue-slider-component";
import SearchBar from "./inputs/SearchBar.vue";
import Switch from "./inputs/Switch.vue";
import Tag from "./inputs/Tag.vue";

export default {
  name: "TopArea",
  components: {
    VueFeather,
    VueSlider,
    SearchBar,
    Switch,
    Tag,
  },
  props: {
    library_name: String,
    section_name: String,
    search_header: Boolean,
    keyLabels: Object,
  },
  data() {
    return {
      keyData: {
        sharps: ["cs", "ds", "x", "fs", "gs", "as"],
        keys: ["c", "d", "e", "f", "g", "a", "b"],
      },
      queryInfo: {
        category: "loops",
        order: ["date", "d"],
        tempo: [0, 200],
        tempoRange: false,
        key: ["", ""],
        date: 0,
        genre: 0,
        filterByKey: false,
        author: "",
      },
      query: {},
      filterEnabled: {
        tempo: false,
        key: false,
        sort: false,
        genre: false,
      },
      filterAdded: {
        tempo: false,
        key: false,
        genre: false,
      },
      filterValues: {
        tempoRange: [0, 200],
        key: ["", false],
      },
      genrelist: [
        ["0", "All Genres"],
        ["56", "8Bit Chiptune"],
        ["52", "Acid"],
        ["3", "Acoustic"],
        ["67", "Afrobeat"],
        ["2", "Ambient"],
        ["66", "Big Room"],
        ["33", "Blues"],
        ["65", "Boom Bap"],
        ["37", "Breakbeat"],
        ["21", "Chill Out"],
        ["36", "Cinematic"],
        ["13", "Classical"],
        ["51", "Comedy"],
        ["44", "Country"],
        ["39", "Crunk"],
        ["17", "Dance"],
        ["55", "Dancehall"],
        ["30", "Deep House"],
        ["23", "Dirty"],
        ["18", "Disco"],
        ["11", "Drum And Bass"],
        ["5", "Dub"],
        ["49", "Dubstep"],
        ["64", "EDM"],
        ["42", "Electro"],
        ["16", "Electronic"],
        ["15", "Ethnic"],
        ["25", "Folk"],
        ["19", "Funk"],
        ["46", "Fusion"],
        ["24", "Garage"],
        ["31", "Glitch"],
        ["43", "Grime"],
        ["53", "Grunge"],
        ["48", "Hardcore"],
        ["61", "Hardstyle"],
        ["27", "Heavy Metal"],
        ["7", "Hip Hop"],
        ["22", "House"],
        ["63", "Indie"],
        ["38", "Industrial"],
        ["6", "Jazz"],
        ["10", "Jungle"],
        ["69", "Latin"],
        ["62", "Lo-Fi"],
        ["57", "Moombahton"],
        ["34", "Orchestral"],
        ["70", "Phonk"],
        ["50", "Pop"],
        ["60", "Psychedelic"],
        ["14", "Punk"],
        ["8", "Rap"],
        ["20", "Rave"],
        ["4", "Reggae"],
        ["32", "Reggaeton"],
        ["45", "Religious"],
        ["12", "RnB"],
        ["1", "Rock"],
        ["29", "Samba"],
        ["41", "Ska"],
        ["59", "Soul"],
        ["47", "Spoken Word"],
        ["9", "Techno"],
        ["28", "Trance"],
        ["54", "Trap"],
        ["58", "Trip Hop"],
        ["68", "UK Drill"],
        ["35", "Weird"],
      ],
    };
  },
  methods: {
    setTempo(values) {
      this.filterAdded.tempo = true;
      this.queryInfo.tempo = values;
    },
    sendQuery() {
      this.$parent.search();
    },
  },
};
</script>
