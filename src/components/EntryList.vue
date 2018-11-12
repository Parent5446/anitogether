<template>
  <!-- Need to wrap all table cells I intend to animate in a div
    because you cannot override table row or cell sizes. -->
  <sui-table
    class="entry-table"
    definition celled compact unstackable selectable
  >
    <sui-table-header>
      <sui-table-row>
        <sui-table-header-cell>Anime</sui-table-header-cell>
        <sui-table-header-cell v-for="user of users"
          :key="user.name"
          collapsing textAlign="right"
        >
          <a v-bind:href="`https://anilist.co/user/${user.name}`" target="_blank">
            <sui-image
              rounded size="mini"
              :src="user.avatar.medium"
              :title="user.name"
            />
          </a>
        </sui-table-header-cell>
        <sui-table-header-cell collapsing textAlign="right">
          Latest
        </sui-table-header-cell>
        <sui-table-header-cell collapsing textAlign="center" class="select-shows">
          <transition name="colfade">
            <div class="animate" v-if="hideSelectActive">
              Show <br> / Hide
            </div>
          </transition>
        </sui-table-header-cell>
      </sui-table-row>
    </sui-table-header>

    <transition-group name="rowfade" tag="tbody">
      <Entry v-for="entry of entries"
        :key="entry.media.id"
        :entry="entry"
        :users="users"
        :hideSelectActive="hideSelectActive"
        v-if="shouldShowEntry(entry)"
      />
    </transition-group>

    <sui-table-footer fullWidth>
      <sui-table-header-cell v-bind:colspan="3 + users.length">
        <sui-button compact size="small" floated="right"
          :primary="hideSelectActive"
          @click="toggleHiddenEntries"
        >
          {{ hideSelectActive
             ? "Hide shows"
             : "Select shows" }}
        </sui-button>
        <span class="stats">
          {{ stats.count }} {{ (users.length > 1) ? "shared" : "" }} anime
          {{ stats.countReleasing > 0 ? `, ${stats.countReleasing} releasing` : "" }}
          {{ stats.hidden > 0 ? `, ${stats.hidden} hidden` : "" }}
          {{ stats.belowThreshold > 0 ? `, ${stats.belowThreshold} below threshold` : "" }}
        </span>
      </sui-table-header-cell>
    </sui-table-footer>
  </sui-table>
</template>

<script>
import Entry from './Entry.vue'

export default {
  components: {
    Entry,
  },
  props: {
    entries: Array,
    users: Array,
    minShared: Number,
    hideSeen: Boolean,
    allShared: Boolean,
  },
  data () {
    return {
      hideSelectActive: false,
    }
  },
  computed: {
    /**
     * Statistics to be rendered in footer.
     * @return {Object}
     */
    stats () {
      const visibleEntries = this.filteredEntries.filter(({media}) => media.visible)
      return {
        count: visibleEntries.length,
        countReleasing: visibleEntries
          .filter(({media}) => media.status === 'RELEASING').length,
        hidden: this.filteredEntries.length - visibleEntries.length,
        belowThreshold: this.entries.length - this.filteredEntries.length,
      }
    },
    /**
     * Filter entries based on criteria (but do not hide).
     * @return {Object}
     */
    filteredEntries () {
      const dstEntries = []
      for (const entry of this.entries) {
        if (this.users.length === 1 || entry.users.size >= Number(this.minShared)) {
          dstEntries.push(entry)
        }
      }
      return dstEntries
    },
  },
  methods: {
    toggleHiddenEntries () {
      this.hideSelectActive = !this.hideSelectActive

      if (!this.hideSelectActive) {
        const hiddenEntries = this.entries
          .filter(({media}) => !media.visible)
          .map(({media}) => media.id)
        this.$emit('updateHidden', hiddenEntries)
      }
    },
    shouldShowEntry (entry) {
      const shareFilter = this.users.length === 1
                          || (
                            this.allShared
                              ? entry.users.size === this.users.length
                              : entry.users.size >= this.minShared)
      const seenFilter = !this.hideSeen
                         || Array.from(entry.users.values()).some(user => user.progress !== entry.media.latestEpisode)
      const selectUiFilter = entry.media.visible || this.hideSelectActive
      return shareFilter && seenFilter && selectUiFilter
    },
  },
}
</script>

<style type="text/css" scoped>
  /*** Sticky table head ***/
  /* Must be on th because of chrome (and edge) bug
   * https://caniuse.com/#feat=css-sticky
   */
  thead th {
    position: sticky;
    top: 0;
    z-index: 5;
  }
  thead th:first-child {
    /* Otherwise background is transparent */
    background: white;
  }

  /*** Columns ***/
  .ui.compact.table th.select-shows {
    padding: 0;
  }
  .select-shows .animate {
    padding: .5em .7em;
  }

  .colfade-enter-active, .colfade-leave-active {
    transition: all 0.5s;
  }

  .colfade-enter, .colfade-leave-to {
    border-width: 0px;
    padding-left: 0;
    padding-right: 0;
    transform: scaleX(0);
    width: 0px;
    /*padding: 0;*/
    /*font-size: 0;*/
    min-width: 0px;
  }

  /*** Hiding and showing entries ***/
  .rowfade-enter-active, .rowfade-leave-active {
    transition: all 0.5s;
  }

  .rowfade-enter, .rowfade-leave-to {
    transform: scaleY(0);
  }

  .rowfade-move {
    transition: transform 1s;
  }
</style>