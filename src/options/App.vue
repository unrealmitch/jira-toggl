<template>
  <div class="container">
    <div class="md-layout">
      <div class="md-layout-item">
        <md-toolbar md-elevation="0">
          <div class="md-layout md-alignment-center-left">
            <img src="/icons/jira-toggl_48.png" alt="Avatar">
            <h3 class="md-title">Toggl To Jira</h3>
            <a href="https://github.com/unrealmitch/jira-toggl#readme" target="_blank" style="position: absolute; right: 20px;font-weight: bold;">
              See Documentation for help
            </a> 
          </div>
        </md-toolbar>
      </div>
    </div>
    <div class="inner-container">
      <div class="md-layout md-gutter">
        <div class="md-layout-item md-size-100">
          <h3>Credentials</h3>
          <md-field>
            <label>Jira Server Url (https://jira.atlassian.net)</label>
            <md-input v-model="jiraUrl" />
          </md-field>
          <md-field>
            <label>Jira email</label>
            <md-input v-model="jiraEmail" />
          </md-field>
          <md-field>
            <label>Toggl API token</label>
            <md-input v-model="togglApiToken" />
          </md-field>
          <br>
          <h3>Options</h3>
          <md-checkbox v-model="jiraMerge">Merge time entries with same comment</md-checkbox>
          <!-- <md-checkbox v-model="allowNumbersInId">Allow numbers in Project ID</md-checkbox> -->
          <md-checkbox v-model="jiraIssueInDescription">Parse Jira issue from description</md-checkbox>
          <md-checkbox v-model="worklogWihtoutDescription">Don't include Issue ID in worklog</md-checkbox><br>
          <md-checkbox v-model="worklogDescriptionSplit">Split worklog description from first occurrence of:</md-checkbox>
          <input v-model="stringSplit" placeholder="Searched string to split" style="contain: content;">
          <br><br>
          <h3>Extra Options</h3>
          <md-checkbox v-model="saveDates">Save dates (Persistent start and end dates)</md-checkbox><br>
          <md-checkbox v-model="weekdayMonday">Start week on monday</md-checkbox><br>
          <md-checkbox v-model="useTogglColors">Use Toggl project colors in issue link</md-checkbox><br>
          <md-checkbox v-model="reverseLogs">Show newest logs at top (logs date in descending order)</md-checkbox><br>

          <br>
          <h3>Jira Options</h3>
          <md-checkbox v-model="getJiraIssueInfo">Get Jira issues info (Show Jira info over Issue link)</md-checkbox><br>
          <md-checkbox v-if="getJiraIssueInfo" v-model="issueStatusFormat">Use issue jira status to set the text color/format of issue link/description</md-checkbox><br>
          <md-checkbox v-if="getJiraIssueInfo" v-model="needConfirmTransition">Request confirmation for issue status change</md-checkbox><br>
          <md-checkbox v-if="getJiraIssueInfo" v-model="allowTransitionByTogglDescription">Change status of issue if toggl description log contains #NameTranstion <i>(like smartcommits, example: TASK-1 #Done)</i></md-checkbox><br>
          <md-field v-if="getJiraIssueInfo">
            <label>Jira Transitions ( Name:Transition ID; Name2:Transition ID2... ) <i>For classic projects use or add the next:</i> Backlog:11; To Do:61; Blocked:71; In Progress: 31; In Review:51; Done: 41;</label>
            <md-input v-model="transitions" />
          </md-field>
          <md-checkbox v-model="clockworkEnabled">Button to Jira Plugin (Usually Clockwork Free)</md-checkbox><br>
          <div v-if="clockworkEnabled">
            <md-field>
              <label>URL Jira Plugin</label>
              <md-input v-model="jiraPlugin" />
            </md-field>
          </div>

          <br>
          <h3>ManicTime Options</h3>
          <md-checkbox v-model="manicTimeEnabled">Enable upload logs to ManicTime Server as Tags</md-checkbox>
          <div v-if="manicTimeEnabled">
            
            <md-field>
              <label>ManicTime Server Url</label>
              <md-input v-model="manicTimeServer" />
            </md-field>
            <md-field>
              <label>ManicTime Token</label>
              <md-input v-model="manicTimeToken" />
            </md-field>
            <md-field>
              <label>ManicTime Timeline of Tags (multiple timelines separate by ,)</label>
              <md-input v-model="manicTimeTimeline" />
            </md-field>
            <md-checkbox v-model="manicTimeAllowRepost">Allow mode to only repost logs in ManicTime Server</md-checkbox><br>
          </div>
          
          <md-button class="md-raised md-accent button__item" @click="saveSettings">
            <span v-show="!isSaving">Save settings</span>
            <span v-show="isSaving">Saving...</span>
          </md-button>

          <md-snackbar :md-active.sync="showSnackbar" md-persistent>
            <span>Your settings have now been saved ✌️</span>
          </md-snackbar>
        </div>
      </div>
    </div>
  </div>
</template>

<script>

export default {
  name: 'App',
  data () {
    return {
      jiraUrl: '',
      jiraEmail: '',
      jiraMerge: false,
      jiraIssueInDescription: true,
      worklogWihtoutDescription: true,
      worklogDescriptionSplit: false,
      allowNumbersInId: true,
      clockworkEnabled: false,
      stringSplit: ':',
      togglApiToken: '',
      isSaving: false,
      showSnackbar: false,
      jiraPlugin: '{jiraUrl}/plugins/servlet/ac/clockwork-free-cloud/clockwork-mywork#!reportName=Toggle2Jira&scope%5BstartingAt%5D={startDate}&scope%5BendingAt%5D={endDate}&selectedBreakdowns%5B%5D=projects&selectedBreakdowns%5B%5D=issues&period=PERIOD_DAY',
      weekdayMonday: true,
      saveDates: false,
      useTogglColors: true,
      getJiraIssueInfo: true,
      issueStatusFormat: true,
      reverseLogs: true,
      transitions: null,
      needConfirmTransition: false,
      allowTransitionByTogglDescription: true,
      manicTimeEnabled: false,
      manicTimeServer: '',
      manicTimeToken: '',
      manicTimeTimeline: '',
      manicTimeAllowRepost: false,
    };
  },
  created () {
    const _self = this;

    browser.storage.sync.get({
      jiraUrl: '',
      jiraEmail: '',
      jiraMerge: false,
      jiraIssueInDescription: true,
      worklogWihtoutDescription: true,
      worklogDescriptionSplit: false,
      allowNumbersInId: true,
      clockworkEnabled: false,
      stringSplit: ':',
      togglApiToken: '',
      jiraPlugin: '{jiraUrl}/plugins/servlet/ac/clockwork-free-cloud/clockwork-mywork#!reportName=Toggle2Jira&scope%5BstartingAt%5D={startDate}&scope%5BendingAt%5D={endDate}&selectedBreakdowns%5B%5D=projects&selectedBreakdowns%5B%5D=issues&period=PERIOD_DAY',
      weekdayMonday: true,
      saveDates: false,
      useTogglColors: true,
      getJiraIssueInfo: true,
      issueStatusFormat: true,
      reverseLogs: true,
      transitions: "To Do:11;In Progress:21;Done:31",
      needConfirmTransition: false,
      allowTransitionByTogglDescription: true,
      manicTimeEnabled: false,
      manicTimeServer: '',
      manicTimeToken: '',
      manicTimeTimeline: '',
      manicTimeAllowRepost: false
    }).then((setting) => {
      _self.jiraUrl = setting.jiraUrl;
      _self.jiraEmail = setting.jiraEmail;
      _self.jiraMerge = setting.jiraMerge;
      _self.jiraIssueInDescription = setting.jiraIssueInDescription;
      _self.worklogWihtoutDescription = setting.worklogWihtoutDescription;
      _self.worklogDescriptionSplit = setting.worklogDescriptionSplit;
      _self.allowNumbersInId = setting.allowNumbersInId;
      _self.clockworkEnabled = setting.clockworkEnabled;
      _self.stringSplit = setting.stringSplit;
      _self.togglApiToken = setting.togglApiToken;
      _self.jiraPlugin = setting.jiraPlugin;
      _self.weekdayMonday = setting.weekdayMonday;
      _self.saveDates = setting.saveDates;
      _self.useTogglColors = setting.useTogglColors;
      _self.getJiraIssueInfo = setting.getJiraIssueInfo;
      _self.issueStatusFormat = setting.issueStatusFormat;
      _self.reverseLogs = setting.reverseLogs;
      _self.transitions = setting.transitions;
      _self.needConfirmTransition = setting.needConfirmTransition;
      _self.allowTransitionByTogglDescription = setting.allowTransitionByTogglDescription;
      _self.manicTimeEnabled = setting.manicTimeEnabled;
      _self.manicTimeServer = setting.manicTimeServer;
      _self.manicTimeToken = setting.manicTimeToken;
      _self.manicTimeTimeline = setting.manicTimeTimeline;
      _self.manicTimeAllowRepost = setting.manicTimeAllowRepost;
    });
  },
  methods: {
    saveSettings () {
      const _self = this;

      _self.isSaving = true;
      browser.storage.sync.set({
        jiraUrl: _self.jiraUrl,
        jiraEmail: _self.jiraEmail,
        jiraMerge: _self.jiraMerge,
        jiraIssueInDescription: _self.jiraIssueInDescription,
        worklogWihtoutDescription: _self.worklogWihtoutDescription,
        worklogDescriptionSplit: _self.worklogDescriptionSplit,
        allowNumbersInId: _self.allowNumbersInId,
        clockworkEnabled: _self.clockworkEnabled,
        stringSplit: _self.stringSplit,
        togglApiToken: _self.togglApiToken,
        jiraPlugin: _self.jiraPlugin,
        weekdayMonday: _self.weekdayMonday,
        saveDates: _self.saveDates,
        useTogglColors: _self.useTogglColors,
        getJiraIssueInfo: _self.getJiraIssueInfo,
        issueStatusFormat: _self.issueStatusFormat,
        reverseLogs: _self.reverseLogs,
        transitions: _self.transitions,
        needConfirmTransition: _self.needConfirmTransition,
        allowTransitionByTogglDescription: _self.allowTransitionByTogglDescription,
        manicTimeEnabled: _self.manicTimeEnabled,
        manicTimeServer: _self.manicTimeServer,
        manicTimeToken: _self.manicTimeToken,
        manicTimeTimeline: _self.manicTimeTimeline,
        manicTimeAllowRepost: _self.manicTimeAllowRepost
      }).then(() => {
        _self.isSaving = false;
        _self.showSnackbar = true;
      });
    }
  }
};
</script>

<style scoped>
  .inner-container {
    padding: 20px;
  }

  .button__item {
    position: fixed;
    bottom: 10px;
    right: 10px;
    background: var(--md-theme-default-accent) !important;
    background-color: var(--md-theme-default-accent)!important;
  }

  img { width: 32px; }
</style>
