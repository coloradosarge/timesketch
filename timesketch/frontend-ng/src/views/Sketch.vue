<!--
Copyright 2019 Google Inc. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

<template>
  <div>
    <!-- Progress indicator when loading sketch data -->
    <v-progress-linear v-if="loadingSketch" indeterminate color="primary"></v-progress-linear>

    <div v-if="sketch.id && !loadingSketch" style="height: 70vh">
      <!-- Empty state -->
      <v-container v-if="!hasTimelines && !loadingSketch" fill-height fluid>
        <v-row align="center" justify="center">
          <v-sheet class="pa-4" style="background: transparent">
            <center>
              <v-img src="/dist/empty-state.png" max-height="100" max-width="300"></v-img>
              <div style="font-size: 2em" class="mb-3 mt-3">It's empty around here</div>
              <ts-upload-timeline-form btn-size="normal" btn-type="rounded"></ts-upload-timeline-form>
            </center>
          </v-sheet>
        </v-row>
      </v-container>

      <!-- Archived state -->
      <v-container v-if="isArchived && !loadingSketch" fill-height fluid>
        <v-row align="center" justify="center">
          <v-sheet class="pa-4">
            <center>
              <v-img src="/dist/empty-state.png" max-height="100" max-width="300"></v-img>
              <div style="font-size: 2em" class="mb-3 mt-3">This sketch is archived</div>
              <v-btn rounded depressed color="primary" @click="unArchiveSketch()"> Bring it back </v-btn>
            </center>
          </v-sheet>
        </v-row>
      </v-container>

      <!-- Rename sketch dialog -->
      <v-dialog v-model="renameSketchDialog" width="600">
        <v-card class="pa-4">
          <ts-rename-sketch @close="renameSketchDialog = false"></ts-rename-sketch>
        </v-card>
      </v-dialog>

      <!-- Top horizontal toolbar -->
      <v-app-bar
        v-if="!loadingSketch"
        app
        clipped-left
        flat
        :color="$vuetify.theme.dark ? '#121212' : 'white'"
        :style="[
          $vuetify.theme.dark
            ? { 'border-bottom': '1px solid hsla(0,0%,100%,.12) !important' }
            : { 'border-bottom': '1px solid rgba(0,0,0,.12) !important' },
        ]"
      >
        <v-btn v-if="hasTimelines && !loadingSketch && !isArchived" icon @click.stop="showLeftPanel = !showLeftPanel">
          <v-icon title="Toggle left panel">mdi-menu</v-icon>
        </v-btn>

        <v-avatar class="ml-n2 mt-1">
          <router-link to="/">
            <v-img src="/dist/timesketch-color.png" max-height="25" max-width="25" contain></v-img>
          </router-link>
        </v-avatar>

        <v-hover v-slot="{ hover }">
          <div class="d-flex flex-wrap">
            <div
              class="flex-1-0"
              @dblclick="renameSketchDialog = true"
              style="
                font-size: 1.1em;
                cursor: pointer;
                white-space: nowrap;
                overflow: hidden;
                text-overflow: ellipsis;
                max-width: 900px;
              "
              :title="sketch.name"
            >
              {{ sketch.name }}
            </div>
            <div>
              <v-icon title="Rename sketch" small class="ml-1" v-if="hover" @click="renameSketchDialog = true"
                >mdi-pencil</v-icon
              >
            </div>
          </div>
        </v-hover>
        <v-spacer></v-spacer>
        <v-btn small rounded depressed v-on:click="switchUI"> Use the old UI </v-btn>

        <!-- Sharing dialog -->
        <v-dialog v-model="shareDialog" width="500">
          <template v-slot:activator="{ on, attrs }">
            <v-btn small rounded depressed color="primary" class="ml-2" v-bind="attrs" v-on="on">
              <v-icon small left>mdi-account-multiple-plus</v-icon>
              Share
            </v-btn>
          </template>
          <ts-share-card @close-dialog="shareDialog = false"></ts-share-card>
        </v-dialog>

        <v-avatar color="grey lighten-1" size="25" class="ml-3">
          <span class="white--text">{{ currentUser | initialLetter }}</span>
        </v-avatar>
        <v-menu offset-y>
          <template v-slot:activator="{ on, attrs }">
            <v-avatar>
              <v-btn small icon v-bind="attrs" v-on="on">
                <v-icon title="Sketch Options">mdi-dots-vertical</v-icon>
              </v-btn>
            </v-avatar>
          </template>
          <v-card>
            <v-list two-line>
              <v-list-item v-if="sketch.user">
                <v-list-item-content>
                  <v-list-item-title>
                    <strong>Created:</strong> {{ sketch.created_at | shortDateTime }}
                  </v-list-item-title>
                  <v-list-item-subtitle>
                    <small>{{ sketch.created_at | timeSince }} by {{ sketch.user.username }}</small>
                  </v-list-item-subtitle>
                </v-list-item-content>
              </v-list-item>

              <v-list-item>
                <v-list-item-content>
                  <v-list-item-title>
                    <strong>Access: </strong>
                    <span v-if="meta.permissions && meta.permissions.public">Public</span>
                    <span v-else>Restricted</span>
                  </v-list-item-title>
                  <v-list-item-subtitle>
                    <small v-if="meta.permissions && meta.permissions.public"
                      >Visible to all users on this server</small
                    >
                    <small v-else>Only people with access can open</small>
                  </v-list-item-subtitle>
                </v-list-item-content>
              </v-list-item>
            </v-list>

            <v-list>
              <v-list-item-group color="primary">
                <v-list-item v-on:click="toggleTheme">
                  <v-list-item-icon>
                    <v-icon>mdi-brightness-6</v-icon>
                  </v-list-item-icon>
                  <v-list-item-content>
                    <v-list-item-title>Toggle theme</v-list-item-title>
                  </v-list-item-content>
                </v-list-item>

                <v-list-item @click="renameSketchDialog = true">
                  <v-list-item-icon>
                    <v-icon>mdi-pencil</v-icon>
                  </v-list-item-icon>
                  <v-list-item-content>
                    <v-list-item-title>Rename sketch</v-list-item-title>
                  </v-list-item-content>
                </v-list-item>

                <v-list-item @click="archiveSketch()" :disabled="isArchived">
                  <v-list-item-icon>
                    <v-icon>mdi-archive</v-icon>
                  </v-list-item-icon>
                  <v-list-item-content>
                    <v-list-item-title>Archive sketch</v-list-item-title>
                  </v-list-item-content>
                </v-list-item>

                <v-list-item v-if="meta.permissions && meta.permissions.delete" @click="deleteSketch()">
                  <v-list-item-icon>
                    <v-icon>mdi-trash-can-outline</v-icon>
                  </v-list-item-icon>
                  <v-list-item-content>
                    <v-list-item-title>Delete sketch</v-list-item-title>
                  </v-list-item-content>
                </v-list-item>

                <a href="/logout/" style="text-decoration: none; color: inherit">
                  <v-list-item>
                    <v-list-item-icon>
                      <v-icon>mdi-logout</v-icon>
                    </v-list-item-icon>

                    <v-list-item-content>
                      <v-list-item-title>Logout</v-list-item-title>
                    </v-list-item-content>
                  </v-list-item>
                </a>
              </v-list-item-group>
            </v-list>
          </v-card>
        </v-menu>
      </v-app-bar>

      <!-- Left panel -->
      <v-navigation-drawer
        v-model="showLeftPanel"
        app
        disable-resize-watcher
        stateless
        clipped
        hide-overlay
        width="410"
      >
        <!-- Dialog for adding a scenario -->
        <v-dialog v-model="scenarioDialog" max-width="500px">
          <v-card>
            <div class="pa-3">
              <h3>Investigative Scenarios</h3>
              <v-select
                v-model="selectedScenario"
                :items="scenarioTemplates"
                item-text="name"
                return-object
                label="Select a scenario"
                outlined
                class="mt-3"
              ></v-select>
              <div v-if="selectedScenario">
                {{ selectedScenario.description }}
              </div>
            </div>
            <v-divider></v-divider>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn @click="scenarioDialog = false" text> Cancel </v-btn>
              <v-btn text color="primary" :disabled="!selectedScenario" @click="addScenario(selectedScenario.id)">
                Add
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>

        <v-tabs v-model="leftPanelTab" grow>
          <v-tab v-for="item in leftPanelTabItems" :key="item"> {{ item }} </v-tab>
        </v-tabs>
        <v-divider></v-divider>
        <v-tabs-items v-model="leftPanelTab">
          <v-tab-item :transition="false">
            <ts-search></ts-search>
            <ts-saved-searches v-if="meta.views"></ts-saved-searches>
            <ts-data-types></ts-data-types>
            <ts-tags></ts-tags>
            <ts-graphs></ts-graphs>
            <ts-stories></ts-stories>
            <ts-intelligence></ts-intelligence>
            <ts-search-templates></ts-search-templates>
            <ts-sigma-rules></ts-sigma-rules>
            <ts-analyzer-results></ts-analyzer-results>
          </v-tab-item>
          <v-tab-item :transition="false">
            <ts-scenario v-for="scenario in activeScenarios" :key="scenario.id" :scenario="scenario"></ts-scenario>
            <v-row class="mt-0 px-2" flat>
              <v-col cols="6">
                <v-card v-if="!Object.keys(scenarioTemplates).length" flat class="pa-4"
                  >No scenarios available yet. Contact your server admin to add scenarios to this server.</v-card
                >
                <v-btn v-else text color="primary" @click="scenarioDialog = true" style="cursor: pointer"
                  ><v-icon left>mdi-plus</v-icon> Add Scenario</v-btn
                >
              </v-col>

              <v-col cols="6">
                <v-btn
                  small
                  text
                  color="primary"
                  v-if="hiddenScenarios.length"
                  @click="showHidden = !showHidden"
                  class="mt-1"
                >
                  <small
                    ><span v-if="showHidden">Hide</span><span v-else>Show</span> hidden scenarios ({{
                      hiddenScenarios.length
                    }})</small
                  >
                </v-btn>
              </v-col>
            </v-row>

            <div v-if="showHidden">
              <ts-scenario v-for="scenario in hiddenScenarios" :key="scenario.id" :scenario="scenario"></ts-scenario>
            </div>
          </v-tab-item>
        </v-tabs-items>
      </v-navigation-drawer>

      <!-- Main (canvas) view -->
      <v-main class="notransition mt-5">
        <router-view
          v-if="sketch.status && hasTimelines && !isArchived"
          @setTitle="(title) => (this.title = title)"
        ></router-view>
      </v-main>

      <!-- Context search -->
      <v-bottom-sheet
        hide-overlay
        persistent
        no-click-animation
        v-model="showTimelineView"
        @click:outside="showTimelineView = false"
        scrollable
      >
        <v-card>
          <v-toolbar dense flat>
            <strong>Context search</strong>
            <v-btn-toggle v-model="contextTimeWindowSeconds" class="ml-10" rounded>
              <v-btn
                v-for="duration in [1, 5, 10, 60, 300, 600, 1800, 3600]"
                :key="duration"
                :value="duration"
                small
                outlined
                @click="updateContextQuery(duration)"
              >
                {{ duration | formatSeconds }}
              </v-btn>
            </v-btn-toggle>
            <v-btn small text class="ml-5" @click="contextToSearch()">Replace search</v-btn>

            <v-spacer></v-spacer>

            <v-btn icon :disabled="timelineViewHeight > 40" @click="increaseTimelineViewHeight()">
              <v-icon>mdi-chevron-up</v-icon>
            </v-btn>
            <v-btn icon :disabled="timelineViewHeight === 0" @click="decreaseTimelineViewHeight()">
              <v-icon>mdi-chevron-down</v-icon>
            </v-btn>
            <v-btn icon @click="showTimelineView = false">
              <v-icon>mdi-close</v-icon>
            </v-btn>
          </v-toolbar>
          <v-divider></v-divider>
          <v-expand-transition>
            <v-card-text :style="{ height: timelineViewHeight + 'vh' }" v-show="!minimizeTimelineView">
              <ts-event-list :query-request="queryRequest" :highlight-event="currentContextEvent"></ts-event-list>
            </v-card-text>
          </v-expand-transition>
        </v-card>
      </v-bottom-sheet>
    </div>
  </div>
</template>

<script>
import ApiClient from '../utils/RestApiClient'
import EventBus from '../main'
import dayjs from '@/plugins/dayjs'

import TsScenario from '../components/Scenarios/Scenario'
import TsSavedSearches from '../components/LeftPanel/SavedSearches'
import TsDataTypes from '../components/LeftPanel/DataTypes'
import TsTags from '../components/LeftPanel/Tags'
import TsSearchTemplates from '../components/LeftPanel/SearchTemplates'
import TsSigmaRules from '../components/LeftPanel/SigmaRules'
import TsIntelligence from '../components/LeftPanel/ThreatIntel'
import TsGraphs from '../components/LeftPanel/Graphs'
import TsStories from '../components/LeftPanel/Stories'
import TsSearch from '../components/LeftPanel/Search'
import TsUploadTimelineForm from '../components/UploadForm'
import TsShareCard from '../components/ShareCard'
import TsRenameSketch from '../components/RenameSketch'
import TsAnalyzerResults from '../components/LeftPanel/AnalyzerResults.vue'
import TsEventList from '../components/Explore/EventList'

export default {
  props: ['sketchId'],
  components: {
    TsScenario,
    TsSavedSearches,
    TsDataTypes,
    TsTags,
    TsSearchTemplates,
    TsSigmaRules,
    TsUploadTimelineForm,
    TsShareCard,
    TsRenameSketch,
    TsIntelligence,
    TsGraphs,
    TsStories,
    TsSearch,
    TsAnalyzerResults,
    TsEventList,
  },
  data() {
    return {
      showSketchMetadata: false,
      navigationDrawer: {
        width: 410,
      },
      selectedScenario: null,
      scenarioDialog: false,
      showLeftPanel: false,
      leftPanelTab: 0,
      leftPanelTabItems: ['EXPLORE', 'INVESTIGATE'],
      renameSketchDialog: false,
      showHidden: false,
      shareDialog: false,
      loadingSketch: false,
      // Context
      timelineViewHeight: 60,
      showTimelineView: false,
      currentContextEvent: {},
      minimizeTimelineView: false,
      queryRequest: {},
      contextStartTime: null,
      contextEndTime: null,
      contextTimeWindowSeconds: 300,
    }
  },
  mounted() {
    this.loadingSketch = true
    this.showLeftPanel = false
    this.$store.dispatch('updateSketch', this.sketchId).then(() => {
      this.$store.dispatch('updateSearchHistory', this.sketchId)
      this.$store.dispatch('updateScenarios', this.sketchId)
      this.$store.dispatch('updateScenarioTemplates', this.sketchId)
      this.$store.dispatch('updateSavedGraphs', this.sketchId)
      this.$store.dispatch('updateGraphPlugins')
      this.$store.dispatch('updateContextLinks')
      this.$store.dispatch('updateAnalyzerList', this.sketchId)
      this.loadingSketch = false
      if (this.hasTimelines && !this.isArchived) {
        this.showLeftPanel = true
      }
    })
    EventBus.$on('showContextWindow', this.showContextWindow)
  },
  beforeDestroy() {
    EventBus.$off('showContextWindow')
  },
  computed: {
    sketch() {
      return this.$store.state.sketch
    },
    meta() {
      return this.$store.state.meta
    },
    isArchived() {
      if (!this.sketch.status || !this.sketch.status.length) {
        return false
      }
      return this.sketch.status[0].status === 'archived'
    },
    scenarios() {
      return this.$store.state.scenarios
    },
    scenarioTemplates() {
      return this.$store.state.scenarioTemplates
    },
    currentUser() {
      return this.$store.state.currentUser
    },
    activeScenarios() {
      if (!this.scenarios) {
        return []
      }
      return this.scenarios.filter((scenario) => !scenario.status.length || scenario.status[0].status === 'active')
    },
    hiddenScenarios() {
      if (!this.scenarios) {
        return []
      }
      return this.scenarios.filter((scenario) => scenario.status.length && scenario.status[0].status === 'hidden')
    },
    hasTimelines() {
      return this.sketch.timelines && this.sketch.timelines.length
    },
    currentRouteName() {
      return this.$route.name
    },
  },
  methods: {
    archiveSketch: function () {
      this.loadingSketch = true
      ApiClient.archiveSketch(this.sketch.id)
        .then((response) => {
          this.$store.dispatch('updateSketch', this.sketch.id).then(() => {
            this.showLeftPanel = false
            this.loadingSketch = false
          })
        })
        .catch((e) => {
          console.error(e)
        })
    },
    unArchiveSketch: function () {
      this.loadingSketch = true
      ApiClient.unArchiveSketch(this.sketch.id)
        .then((response) => {
          this.$store.dispatch('updateSketch', this.sketch.id).then(() => {
            this.loadingSketch = false
            this.showLeftPanel = true
          })
        })
        .catch((e) => {
          console.error(e)
        })
    },
    deleteSketch: function () {
      if (confirm('Are you sure you want to delete the sketch?')) {
        ApiClient.deleteSketch(this.sketch.id)
          .then((response) => {
            this.$router.push({ name: 'Home' })
          })
          .catch((e) => {
            console.error(e)
          })
      }
    },
    generateContextQuery(event) {
      let timestampMillis = this.$options.filters.formatTimestamp(event._source.timestamp)
      this.contextStartTime = dayjs.utc(timestampMillis).subtract(this.contextTimeWindowSeconds, 'second')
      this.contextEndTime = dayjs.utc(timestampMillis).add(this.contextTimeWindowSeconds, 'second')
      let startChip = {
        field: '',
        value: this.contextStartTime.toISOString() + ',' + this.contextEndTime.toISOString(),
        type: 'datetime_range',
        operator: 'must',
        active: true,
      }
      let queryFilter = {
        from: 0,
        terminate_after: 40,
        size: 40,
        indices: ['_all'],
        order: 'asc',
        chips: [startChip],
      }
      let queryRequest = { queryString: '*', queryFilter: queryFilter }
      return queryRequest
    },
    updateContextQuery(duration) {
      this.contextTimeWindowSeconds = duration
      this.queryRequest = this.generateContextQuery(this.currentContextEvent)
    },
    contextToSearch() {
      let queryRequest = this.generateContextQuery(this.currentContextEvent)
      queryRequest.doSearch = true
      EventBus.$emit('setQueryAndFilter', queryRequest)
      this.showTimelineView = false
    },
    showContextWindow(event) {
      this.currentContextEvent = event
      this.queryRequest = this.generateContextQuery(event)
      this.showTimelineView = true
    },
    increaseTimelineViewHeight: function () {
      this.minimizeTimelineView = false
      if (this.timelineViewHeight > 70) {
        return
      }
      this.timelineViewHeight += 30
    },
    decreaseTimelineViewHeight: function () {
      this.minimizeTimelineView = false
      if (this.timelineViewHeight < 50) {
        this.minimizeTimelineView = true
        this.timelineViewHeight = 0
        return
      }
      this.timelineViewHeight -= 30
    },
    closeTimelineView: function () {
      this.minimizeTimelineView = true
      this.timelineViewHeight = 0
    },

    toggleTheme: function () {
      this.$vuetify.theme.dark = !this.$vuetify.theme.dark
      localStorage.setItem('isDarkTheme', this.$vuetify.theme.dark.toString())
      let element = document.body
      element.dataset.theme = this.$vuetify.theme.dark ? 'dark' : 'light'
    },
    switchUI: function () {
      window.location.href = window.location.href.replace('/sketch/', '/legacy/sketch/')
    },
    addScenario: function (scenarioId) {
      this.scenarioDialog = false
      ApiClient.addScenario(this.sketch.id, scenarioId)
        .then((response) => {
          this.$store.dispatch('updateScenarios', this.sketch.id)
        })
        .catch((e) => {})
    },
  },
  watch: {
    hasTimelines(newVal, oldVal) {
      if (oldVal === 0 && newVal > 0) {
        this.showLeftPanel = true
      }
      if (oldVal > 0 && newVal === 0) {
        this.showLeftPanel = false
      }
    },
  },
}
</script>
