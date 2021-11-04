<template>
  <div class="container">
    <div class="md-layout">
      <div class="md-layout-item">
        <md-toolbar md-elevation="0">
          <div class="md-layout md-alignment-center-left">
            <img src="/icons/jira-toggl_48.png" alt="Avatar">
            <h3 class="md-title">Toggl To Jira</h3>
          </div>
          <div class="md-toolbar-section-end">
            <a v-if="clockworkEnabled" :href="clockworkUrl()" target="_blank">
              <md-button class="md-icon-button">
                <md-icon>timer</md-icon>
              </md-button>
            </a>
            <md-button class="md-icon-button" @click="refreshEntries">
              <md-icon>autorenew</md-icon>
            </md-button>
            <md-button @click="toggleTheme" class="md-icon-button">
                <md-icon v-if="this.theme == 'darkMode'">light_mode</md-icon>
                <md-icon v-else>dark_mode</md-icon>     
            </md-button>
            <a href="../options/options.html" target="_blank">
              <md-button class="md-icon-button">
                <md-icon>settings</md-icon>
              </md-button>
            </a>
          </div>
        </md-toolbar>
      </div>
    </div>
    <div class="button-reset-float">
      <u @click="moveToday">Reset dates</u>
    </div>

    <div class="inner-container">
      <div class="md-layout md-gutter">
        <md-button :disabled="blockFetch" class="md-icon-button material-icons button-navigation-calendar" @click="dayMinus">
          <md-icon>navigate_before</md-icon>
        </md-button>

        <div class="md-layout-item">
          <span class="datepicker-label md-caption">Start date</span>
          <md-datepicker v-model="startDate" :readonly="blockFetch" md-immediately />
        </div>
        <div class="md-layout-item">
          <span class="datepicker-label md-caption">End date</span>
          <md-datepicker v-model="endDate" :readonly="blockFetch" md-immediately />
        </div>
        <md-button :disabled="blockFetch" class="md-icon-button material-icons button-navigation-calendar" @click="dayPlus">
          <md-icon>navigate_next</md-icon>
        </md-button>
      </div>
      <div class="md-layout" :class="{ 'disabled-div': isSaving }">
        <div class="md-layout-item">
          <md-table>
            <md-table-row>
              <md-table-head>
                <md-checkbox v-model="syncAllLogs" class="custom-checkbox" @change="doSyncAllLogs" />
              </md-table-head>
              <md-table-head>Issue</md-table-head>
              <md-table-head>Description</md-table-head>
              <md-table-head>Date</md-table-head>
              <md-table-head>Duration</md-table-head>
              <md-table-head>Logged</md-table-head>
            </md-table-row>

            <md-table-row v-for="log in logs" :key="log.id">
              <md-table-cell class="no-wrap">
                <md-checkbox v-if="!allowRepostManicTime && (log.issue === 'NO ID' || log.isSynced || log.duration < 60)" 
                  v-model="checkedLogs" disabled :value="log" />
                <md-checkbox v-else v-model="checkedLogs" :value="log" />
              </md-table-cell>
              <md-table-cell class="no-wrap">
                <div class="tooltip">
                  <a v-if="log.issue != 'NO ID'" :href="jiraUrl + '/browse/' + log.issue" target="_blank" :style="{color:getLogColor(log)}">
                  {{ log.issue }}</a>
                  <a v-else>{{ log.issue }}</a>
                  <span v-if="showJiraIssueInfo" class="tooltiptextright tooltiptext">
                    <p><span v-html="getIssueInfo(log)"></span></p>
                  </span>
                </div>
              </md-table-cell>
              <md-table-cell>
                <div class="tooltip">
                  <p><span v-html="getLogDescriptionPanel(log)"></span></p>
                  <span class="tooltiptext tooltiptext-top tooltiptextmax">{{getProjectInfo(log)}}</span>
                </div>
              </md-table-cell>
              <md-table-cell class="no-wrap">
                {{
                  $moment(log.start).format("l")
                }}
              </md-table-cell>
              <md-table-cell class="no-wrap" :class="log.duration > 60 ? 'timeBlack' : 'timeRed'">
                {{ formatDuration(log.duration) }}
              </md-table-cell>
              <md-table-cell class="no-wrap">
                <md-icon v-show="log.isSynced" class="md-accent">check_circle</md-icon>
                <md-icon v-show="log.isSyncedManicTime" class="md-accent manicTimeIconChecked">check_circle</md-icon>
              </md-table-cell>
            </md-table-row>

            <md-table-row>
              <md-table-cell class="no-wrap" />
              <md-table-cell />
              <md-table-cell class="no-wrap"><b>TOTAL</b></md-table-cell>
              <md-table-cell class="no-wrap">
                <div class="tooltip">
                  <p><i>{{ totalDuration(true) }}</i></p>
                  <span class="tooltiptext-top tooltiptext">Time in Jira</span>
                </div>
              </md-table-cell>
              <md-table-cell class="no-wrap">
                <div class="tooltip">
                  <p>{{ totalDuration() }}</p>
                  <span class="tooltiptext-top tooltiptext">Time in Toggl</span>
                </div>
              </md-table-cell>
              <md-table-cell class="no-wrap" />
            </md-table-row>
          </md-table>
        </div>
      </div>
      <div class="button__container">
        <md-button v-if="checkedLogs.length && !isSaving" class="md-raised md-accent" @click="syncToJira">
          <span v-show="!isSaving">Log work</span>
        </md-button>
        <md-button v-else disabled class="md-raised md-accent" @click="syncToJira">
          <span v-show="!isSaving">Log work</span>
          <span v-show="isSaving">Logging...</span>
        </md-button>
      </div>
      <div v-show="manicTimeEnabled && manicTimeAllowRepost" class="button-repost-float">
        <u @click="alternateRepostManicime">{{ allowRepostManicTime ? 'Hide ' : 'Only ' }} repost ManicTime</u>
      </div>
      <md-snackbar v-if="!errorMessage" :md-active.sync="showSnackbar" md-persistent>
        <span>Yay! Your entries has been logged to Jira✌️</span>
      </md-snackbar>
    </div>
    <md-toolbar v-if="errorMessage" class="md-accent error-message md-layout md-alignment-center-center">
      <p class="error-msg">{{ errorMessage }}</p>
    </md-toolbar>
  </div>
</template>

<script>
import axios from 'axios';
import moment from 'moment';

const initalStartDate = new Date(moment().startOf('day'));
const initalEndDate = new Date(moment().endOf('day'));

export default {
  data () {
    return {
      checkedLogs: [],
      syncAllLogs: false,
      startDate: initalStartDate,
      endDate: initalEndDate,
      logs: [],
      projectsToggl: [],
      worklogsJira: [],
      issuesJira: [],
      errorMessage: null,
      jiraUrl: '',
      jiraEmail: '',
      jiraMerge: true,
      jiraIssueInDescription: true,
      worklogWihtoutDescription: true,
      worklogDescriptionSplit: true,
      allowNumbersInId: true,
      clockworkEnabled: false,
      stringSplit: ':',
      togglApiToken: '',
      isSaving: false,
      showSnackbar: false,
      blockFetch: false,
      weekdayMonday: true,
      saveDates: false,
      useTogglColors: true,
      showJiraIssueInfo: false,
      reverseLogs: true,
      theme: '',
      manicTimeEnabled: false,
      manicTimeServer: '',
      manicTimeToken: '',
      manicTimeTimeline: '',
      allowRepostManicTime: false,
    };
  },

  watch: {
    startDate: function (newVal, oldVal) {
      if (newVal.toString() !== oldVal.toString()) {
        this.refreshEntries();
      }
    },
    endDate: function (newVal, oldVal) {
      if (newVal.toString() !== oldVal.toString()) {
        this.refreshEntries();
      }
    }
  },

  created () {
    const _self = this;

    browser.storage.sync
      .get({
        jiraUrl: '',
        jiraEmail: '',
        jiraMerge: true,
        jiraIssueInDescription: true,
        worklogWihtoutDescription: true,
        worklogDescriptionSplit: true,
        allowNumbersInId: true,
        clockworkEnabled: false,
        stringSplit: ':',
        togglApiToken: '',
        jiraPlugin: '',
        weekdayMonday: true,
        startDate: initalStartDate,
        endDate: initalEndDate,
        saveDates: false,
        useTogglColors: true,
        showJiraIssueInfo: false,
        reverseLogs: true,
        manicTimeEnabled: false,
        manicTimeServer: '',
        manicTimeToken: '',
        manicTimeTimeline: '',
        manicTimeAllowRepost: false,
      })
      .then((setting) => {
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
        _self.showJiraIssueInfo = setting.showJiraIssueInfo;
        _self.reverseLogs = setting.reverseLogs;
        if (_self.saveDates) {
          _self.startDate = setting.startDate;
          _self.endDate = setting.endDate;
        }
        this.$material.locale.firstDayOfAWeek = _self.weekdayMonday;
        _self.manicTimeEnabled = setting.manicTimeEnabled;
        _self.manicTimeServer = setting.manicTimeServer;
        _self.manicTimeToken = setting.manicTimeToken;
        _self.manicTimeTimeline = setting.manicTimeTimeline;
        _self.manicTimeAllowRepost = setting.manicTimeAllowRepost;
      });
  },
  
  mounted() {
    let localTheme = localStorage.getItem('theme'); //gets stored theme value if any
    document.documentElement.setAttribute('data-theme', localTheme); // updates the data-theme attribute
    this.theme = localTheme;
  },

  methods: {
    refreshEntries () {
      if (this.saveDates) {
        this.saveActualDates();
      }
      this.checkedLogs = [];
      this.logs = [];
      this.projectsToggl = [];
      this.worklogsJira = [];
      this.issuesJira = [];
      this.fetchEntries();
    },

    processJiraDescription (description) {
      const _self = this;
      if (_self.worklogDescriptionSplit && _self.stringSplit) {
        if (description.includes(_self.stringSplit)) {
          return description
            .split(_self.stringSplit)
            .slice(1)
            .join(_self.stringSplit);
        }
      }

      return description;
    },

    async syncToJira () {
      const _self = this;
      const headers = {
        'X-Atlassian-Token': 'no-check'
      };

      _self.blockFetch = true;
      _self.isSaving = true;
      const awaitingIssues = {};
      for (let log of this.checkedLogs) {
        if(!_self.manicTimeEnabled || !_self.allowRepostManicTime){
          if (log.issue in awaitingIssues) {
            await awaitingIssues[log.issue];
          }
          const promise = axios({
            method: 'post',
            url:
              _self.jiraUrl + '/rest/api/latest/issue/' + log.issue + '/worklog',
            data: {
              timeSpentSeconds: log.duration,
              comment: _self.processJiraDescription(
                _self.worklogWihtoutDescription
                  ? log.description.replace(log.issue, '')
                  : log.description
              ),
              started: _self.toJiraDateTime(log.start)
            },
            headers: headers
          })
            .then(function (response) {
              // _self.isSaving = false;
              // _self.showSnackbar = true;
              _self.checkIfAlreadyLogged(log);
              // _self.checkedLogs = [];
              // _self.syncAllLogs = false;
            })
            .catch(function (error) {
              _self.errorMessage = error;
            })
            .finally(function () {
              delete awaitingIssues[log.issue];
            });
          awaitingIssues[log.issue] = promise;

          if (!_self.jiraMerge) {
            await promise;
          }
        }

        if(_self.manicTimeEnabled)
            await _self.pushLogsToManicTime(log);
      }

      _self.blockFetch = false;
      _self.isSaving = false;
      _self.showSnackbar = true;
    },

    toJiraDateTime (date) {
      let parsedDate = Date.parse(date);
      let jiraDate = Date.now();
      if (parsedDate) {
        const offset = new Date().getTimezoneOffset();
        jiraDate = new Date(parsedDate - offset * 60 * 1000);
      }
      let dateString = jiraDate.toISOString();
      let timeZoneString;
      timeZoneString = new Date().toString().match(/([-+][0-9]+)\s/)[1];
      dateString = dateString.replace('Z', timeZoneString);
      return dateString;
    },

    doSyncAllLogs () {
      let _self = this;
      if (this.syncAllLogs) {
        _self.logs.forEach(function (log) {
          if (log.issue !== 'NO ID' && !log.isSynced) {
            _self.checkedLogs.push(log);
          }
        });
      } else {
        this.checkedLogs = [];
      }
    },

    formatDuration (duration) {
      duration = Number(duration);

      if (duration < 0) {
        return 'WIP';
      }

      if (duration < 60) {
        return 'Too short';
      }

      let h = Math.floor(duration / 3600);
      let m = Math.floor((duration % 3600) / 60);
      let hDisplay = h > 0 ? h + 'h' : '';
      let mDisplay = m > 0 ? m + 'm' : '';
      return hDisplay + ' ' + mDisplay;
    },

    isSameStart (worklog, log) {
      return new Date(worklog.started).getTime() === new Date(log.start).getTime();
    },

    async checkIfAlreadyLogged (log) {
      const _self = this;
      _self.worklogsJira.forEach(function (worklogJira) {
        if (log.issue === worklogJira.issueID) {
          worklogJira.worklogs.forEach(function (worklog) {
            _self.findLogInWorklogs(log, worklog);
          });
          return;
        }
      });

      await axios
        .get(_self.jiraUrl + '/rest/api/latest/issue/' + log.issue + '/worklog')
        .then(function (response) {
          let worklogs = response.data.worklogs;
          response.data.issueID = log.issue;
          _self.worklogsJira.push(response.data);
          worklogs.forEach(function (worklog) {
            _self.findLogInWorklogs(log, worklog);
          });
        });
    },

    findLogInWorklogs(log, worklog){
      const _self = this;
      if (
        _self.isSameStart(worklog, log) &&
        worklog.author.emailAddress?.toLowerCase() === _self.jiraEmail?.toLowerCase()
      ){
        let logIndex = _self.logs.findIndex((i) => i.id === log.id);
        if (typeof _self.logs[logIndex] !== 'undefined') {
          _self.logs[logIndex].isSynced = true;
          return;
        }
      }
    },

    matchIssueId (name) {
      if (this.allowNumbersInId) {
        return name.match(/^[A-Z][A-Z,0-9]*-[0-9]*/);
      } else {
        return name.match(/^[A-Z]*-[0-9]*/);
      }
    },

    getIssue (log) {
      let _self = this;
      return new Promise(function (resolve, reject) {
        if (_self.jiraIssueInDescription && log.description != null) {
          const parsedIssue = _self.matchIssueId(log.description);
          if (parsedIssue) {
            resolve(parsedIssue[0]);
          }
          // reject(log);  If don't find in description, search in project title
        } else if (log.description == null) {
          log.description = ''; // Set empty string (not null), to avoid reference null problems
        }
        if (typeof log.pid !== 'undefined') {
          _self.projectsToggl.forEach(function (projectToggl) {
            if (log.pid === projectToggl.id) {
              const parsedIssue = _self.matchIssueId(issue.data.data.name);
              if (parsedIssue) {
                log.projectID = parsedIssue[0];
                log.projectData = projectToggl;
                resolve(parsedIssue[0]);
              } else {
                log.projectID = null;
                reject(log);
              }
              return;
            }
          });

          axios.get('https://api.track.toggl.com/api/v8/projects/' + log.pid, {
            headers: {
              'Content-Type': 'application/json',
              'Authorization':
                'Basic ' + btoa(_self.togglApiToken + ':api_token')
            }
          }).then(function (issue) {
              _self.projectsToggl.push(issue.data.data);
              const parsedIssue = _self.matchIssueId(issue.data.data.name);
              if (parsedIssue) {
                log.projectID = parsedIssue[0];
                log.projectData = issue.data.data;
                resolve(parsedIssue[0]);
              } else {
                log.projectID = null;
                reject(log);
              }
          });
        } else {
          reject(log);
        }
      });
    },

    getDescriptionID(log){
      if(log.description != null && log.description != ''){
        const parsedIssue = _self.matchIssueId(log.description);
        if (parsedIssue) {
          return parsedIssue[0];
        }
      }
      return null;
    },

    getProjectID(log){
      if (typeof log.pid !== 'undefined') {
        axios
          .get('https://api.track.toggl.com/api/v8/projects/' + log.pid, {
            headers: {
              Authorization:
                'Basic ' + btoa(_self.togglApiToken + ':api_token')
            }
          })
          .then(function (issue) {
            const parsedIssue = _self.matchIssueId(issue.data.data.name);
            if (parsedIssue) {
              resolve(parsedIssue[0]);
            } else {
              reject(log);
            }
          });
      } else {
        reject(log);
      }
    },

    async fetchEntries () {
      let _self = this;
      const offset = new Date().getTimezoneOffset();
      const sign = offset <= 0 ? '+' : '-';
      function abspad (num) { return ('0' + Math.abs(num)).slice(-2); }
      const timezone = `${sign}${abspad(offset / 60)}:${abspad(offset % 60)}`;

      let startDate = moment(this.startDate)
        .utc(true)
        .toISOString(true)
        .replace('+00:00', timezone);
      let endDate = moment(this.endDate)
        .add(1, 'days')
        .utc(true)
        .toISOString(true)
        .replace('+00:00', timezone);

      if (_self.blockFetch) {
        return;
      }

      _self.blockFetch = true;
      await axios
        .get('https://api.track.toggl.com/api/v8/time_entries', {
          headers: {
            Authorization: 'Basic ' + btoa(_self.togglApiToken + ':api_token')
          },
          params: {
            start_date: startDate,
            end_date: endDate
          }
        })
        .then(async function (entries) {
          if(_self.reverseLogs)
            entries.data.reverse();

          // await entries.data.forEach(async function (log) {
          for (let i = 0; i < entries.data.length; i++) {
            let log = entries.data[i];
            await _self
              .getIssue(log)
              .then(async function (issueName) {
                let logObject = log;
                logObject.isSynced = false;
                logObject.isSyncedManicTime = false;
                logObject.issue = issueName;
                logObject.checked = '';

                if (_self.jiraMerge) {
                  let logIndex = _self.logs.findIndex(
                    (i) => i.description === log.description && i.issue === issueName
                  );
                  if (logIndex !== -1) {
                    _self.logs[logIndex].duration =
                      _self.logs[logIndex].duration + logObject.duration;
                  } else {
                    _self.logs.push(logObject);
                  }
                } else {
                  _self.logs.push(logObject);
                }

                _self.checkIfAlreadyLogged(log);

                if(_self.showJiraIssueInfo)
                  await _self.getIssueFromJira(log);
              })
              .catch(function (log) {
                // There is no ID for the entry but we still need to print it out to the user
                let logObject = log;
                logObject.isSynced = false;
                logObject.isSyncedManicTime = false;
                logObject.issue = 'NO ID';
                logObject.checked = '';
                _self.logs.push(logObject);
              });

              await new Promise(resolve => setTimeout(resolve, 50));  //Delay to avoid to many requests (error 429)
          } // });
          _self.blockFetch = false;
        })
        .catch(function (error) {
          _self.blockFetch = false;
          if (
            typeof error.response !== 'undefined' &&
            error.response.status === 403
          ) {
            _self.errorMessage = 'Please add your Toggl API token';
          } else {
            _self.errorMessage =
              typeof error.response !== 'undefined'
                ? error.response.statusText
                : error.response;
          }
        });
    },

    totalDuration (synced = false) {
      let _self = this;
      if (!_self.logs.length) {
        return _self.blockFetch ? 'Loading...' : 'No entries!';
      }

      let totalDuration = 0;
      _self.logs.forEach(function (log) {
        if (log.duration && log.duration > 0) {
          if (!synced || log.isSynced) {
            totalDuration += log.duration;
          }
        }
      });
      if (totalDuration) {
        return _self.formatDuration(totalDuration);
      }
    },

    dayMinus () {
      if (this.blockFetch) {
        return;
      }
      this.moveDays(-1);
    },

    dayPlus () {
      if (this.blockFetch) {
        return;
      }
      this.moveDays(1);
    },

    moveDays (ndays) {
      let newStartDate = moment(this.startDate).add(ndays, 'days');
      let newEndDate = moment(this.endDate).add(ndays, 'days');
      this.startDate = new Date(newStartDate.startOf('day'));
      this.endDate = new Date(newEndDate.startOf('day'));
      this.refreshEntries();
    },

    moveToday () {
      this.startDate = new Date(moment().startOf('day'));
      this.endDate = this.startDate;
      this.refreshEntries();
    },

    clockworkUrl () {
      let startDate = moment(this.startDate)
        .utc(true)
        .toISOString(true)
        .slice(0, 10);
      let endDate = moment(this.endDate)
        .utc(true)
        .toISOString(true)
        .slice(0, 10);
      return (
        this.jiraPlugin.replace('{jiraUrl}', this.jiraUrl).replace('{startDate}', startDate).replace('{endDate}', endDate)
      );
    },

    formatDateToPicker (date) {
      const y = new Date(date).getFullYear();
      const m = new Date(date).getMonth() + 1;
      const d = new Date(date).getDate();
      return y.toString() + '-' + m.toString() + '-' + d.toString();
    },

    saveActualDates () {
      const _self = this;
      _self.isSaving = true;
      browser.storage.sync.set({
        startDate: this.formatDateToPicker(_self.startDate),
        endDate: this.formatDateToPicker(_self.endDate)
      }).then(() => {
        _self.isSaving = false;
      });
    },

    getLogDescriptionPanel(log){
      if(log.description && log.description.length > 0){
        return "<p>" + log.description + "</p>";
      }else if(log.projectData != null && log.projectData.name && log.projectData.name.length > 0){
        return '<i><p style="opacity:60%;">' + log.projectData.name + "</p></i>";
      }else{
        return '<i><p style="opacity:50%;"><i>No Description</p></i>';
      }
    },

    getLogColor(log){
      if(this.useTogglColors && log.projectData && log.projectData.hex_color){
        return log.projectData.hex_color;
      }else{
        return "";
      }
    },

    getProjectInfo(log){
      if(log.projectData != null && log.projectData.name && log.projectData.name.length > 0){
        if(log.projectData.actual_hours)
          return log.projectData.name + " [" + log.projectData.actual_hours + "h]";
        else
          return log.projectData.name;
      }else{
        return "No Project Info";
      }
    },

    getIssuesJira(){
      let _self = this;
      if(_self.showJiraIssueInfo)
        _self.logs.forEach(function (log) {
          if (log.issue && log.issue !== '') {
            _self.getIssueFromJira(log);
          }
        });
    },

    async getIssueFromJira(log){
      const _self = this;
      let id = log.issue;
      if( (id == null || id == '') && log.projectID != null && log.projectID != '')
        id = log.projectID;

      if( id == null || id == '' || id == "NO ID" )
        return;

      _self.issuesJira.forEach(function (issueJira) {
        if (id === issueJira.key) {
          log.issueJira = issueJira;
          return;
        }
      });

      await axios
        .get(_self.jiraUrl + '/rest/api/latest/issue/' + log.issue)
        .then(function (response) {
          let issueJira = response.data;
          _self.worklogsJira.push(issueJira);
          log.issueJira = issueJira;
          let logIndex = _self.logs.findIndex((i) => i.id === log.id);
          if (typeof _self.logs[logIndex] !== 'undefined') {
            _self.logs[logIndex].issueJira = issueJira;
          }
        });
    },

    getIssueInfo(log){
      if(log.issueJira){
        let output = "<i>" + log.issueJira.fields.summary + "</i><br>";
        try{
          output += log.issueJira.fields.issuetype.name + " - ";
          output += "<b>" + log.issueJira.fields.status.name + "</b> - ";
          output += log.issueJira.fields.priority.name + "<br>";
          output += (log.issueJira.fields.timespent/3600).toFixed(2) + "h";
          if(log.issueJira.fields.timeestimate && log.issueJira.fields.timeestimate > 0)
            output += " / " + (log.issueJira.fields.timeestimate/3600).toFixed(2) + "h";
          output += " (" +  log.issueJira.fields.progress.percent + "%)<br>";
          if(log.issueJira.fields.assignee){
            output += log.issueJira.fields.assignee.displayName;
          }else{
            output += "Unassigned";
          }
            
        }catch{

        }

        return output;
      }else{
        return "No Issue Info";
      }
    },

    toggleTheme() {
      this.theme = this.theme == 'darkMode' ? '' : 'darkMode';
      document.documentElement.setAttribute('data-theme', this.theme);
      localStorage.setItem('theme', this.theme);
    },

    async pushLogsToManicTime(log){
      const _self = this; 
      if(!log || log.isSyncedManicTime)
        return;
      const projectID = log && log.projectID != null ? log.projectID : "No Project";
      const issueID = log && log.projectID != log.issue && log.issue != '' ? (', ' + log.issue) : '';
      const description = log.description;
      const start = _self.toJiraDateTime(log.start);
      const duration = log.duration;
      const data ={
        values: {
          name: "Toggl, " + projectID + issueID,
          notes: description,
          timeInterval: {
              start:  start,
              duration: duration
          }
        }
      };
      const timelines = _self.manicTimeTimeline.split(',');

      for(let i = 0; i < timelines.length; i++){
        await _self.pushLogToManicTime(log, timelines[i], data);
      }
    },

    async pushLogToManicTime(log, timeline, data, retry = false){
      const _self = this;
      const url = _self.manicTimeServer +  "/api/timelines/" + timeline.trim() + "/activities";
      const headers = { 
        'Authorization': 'Bearer ' + _self.manicTimeToken, 
        'Content-Type': 'application/json'
      };
      const promise = axios.post(url, data, {headers: headers})
      .then(function (response) {
        log.isSyncedManicTime = true;
      })
      .catch(function (error) {
        log.isSyncedManicTime = false;
        console.log(error);
        if(retry){
          _self.pushLogToManicTimeUnique(log, false);
        }
      });

      return promise;
    },

    alternateRepostManicime(log, data, retry = true){
      this.allowRepostManicTime = !this.allowRepostManicTime;
    }

  }
};
</script>

<style>
.container {
  position: relative;
  background: #fff;
  width: 700px;
  height: 600px;
  overflow: auto;
}
.md-layout-item {
  position: relative;
}

.inner-container {
  padding: 20px;
  background: #fff;
}

.md-table-cell-container,
.md-table-head-label {
  padding: 0 5px;
}

.md-checkbox {
  margin: 0;
}

.datepicker-label {
  position: absolute;
  /* left: 44px; */
  margin-left: 35.7px;
  font-size: 12px;
}

.md-table-head-container {
  padding: 4px 0;
  height: auto;
  text-align: left;
}

.md-table-row:hover:not(.md-header-row) .md-table-cell {
  background: unset !important;
}

.no-wrap {
  white-space: nowrap;
}

.button__container {
  display: flex;
  justify-content: flex-end;
}

.md-button {
  margin-left: 0;
}

.error-message {
  text-align: center;
  font-size: 16px;
  position: absolute;
  bottom: 0;
}

.error-msg{
    color: white !important;
}

.md-field.md-focused .md-input,
.md-field.md-focused .md-textarea,
.md-field.md-has-value .md-input,
.md-field.md-has-value .md-textarea {
  font-size: 14px;
}
.custom-checkbox {
  margin-top: 4px;
}

img {
  width: 32px;
}

.timeRed {
  color: var(--md-theme-default-accent, rgb(255, 0, 0));
  font-weight: bold;
}

.timeBlack {
  color: black;
}

.button-navigation-calendar {
  min-width: 20px;
  margin-top: 15px;
}

.button-reset-float{
  position: absolute;
  float: right;
  display: inline;
  overflow: hidden;
  right: 5px;
  margin-top: 5px;
}

.md-toolbar.md-theme-default.md-accent{
  position: sticky;
  bottom: 0;
  z-index: 10;
}

/* ToolTip */

.tooltip {
  position: relative;
  display: inline-block;
  border-bottom: 1px dotted black;
}

.tooltip .tooltiptext {
  visibility: hidden;
  width: 150px;
  background-color: #555;
  color: #fff;
  text-align: center;
  border-radius: 6px;
  padding: 5px 0;
  position: absolute;
  z-index: 1;
  opacity: 0;
  transition: opacity 0.3s;
}

.tooltiptext-top {
  bottom: 125%;
  left: 50%;
  margin-left: -75px;
}

.tooltip .tooltiptextmax{
  width: 300px;
  margin-left: -150px;
  padding: 2px 0;
  left: 150px;
  bottom: 95%;
}

.tooltip .tooltiptextright {
  top: -5%;
  left: 120%;
  width: 325px;
  white-space: pre-line;
  padding: 1px 5px;
}

.tooltip .tooltiptext::after {
  content: "";
  position: absolute;
  top: 100%;
  left: 50%;
  margin-left: -5px;
  border-width: 5px;
  border-style: solid;
  border-color: #555 transparent transparent transparent;
}

.tooltiptextright::after {
  content: "";
  position: absolute;

  right: 100%;
  margin-top: -5px;
  border-width: 5px;
  border-style: solid;
}

.tooltip:hover .tooltiptext {
  visibility: visible;
  opacity: 1;
}

.button-repost-float{
  float: left;
  display: inline;
  overflow: hidden;
  left: 0px;
  bottom: 0px;
  position: relative;
  font-size: 12px;
}

.manicTimeIconChecked {
  color: yellow !important;
  min-width: 5px;
  width: 15px;
  font-size: 14px !important;
}

/* Theme Part */

body{
  color: var(--md-theme-default-primary, black);  
}

* {
  color: var(--md-theme-default-primary, black);  
}

table {
  background-color: var(--md-theme-default-background, white);
}

.md-icon{
  /* -webkit-text-fill-color: var(--md-theme-default-background-3, black); */
}

.md-icon.md-theme-default.md-icon-image svg{
  fill: var(--md-theme-default-primary, black);
}

.md-icon.md-theme-default.md-icon-font{
  color: var(--md-theme-default-primary, black);  
}

svg{
  fill: var(--md-theme-default-primary, black);
}

.md-input, .md-field.md-theme-default.md-has-value .md-textarea{
  -webkit-text-fill-color: var(--md-theme-default-background-3, black) !important;
}

.md-field.md-theme-default:after{
  background-color: var(--md-theme-default-background-4, black);
}

.md-checkbox.md-theme-default .md-checkbox-container{
  border-color: var(--md-theme-default-background-4, black);
}

.md-checkbox.md-theme-default.md-disabled .md-checkbox-container{
  border-color: var(--md-theme-default-background-5, black);
}

.inner-container {
  background-color: var(--md-theme-default-background, white);
}
.container {
  background-color: var(--md-theme-default-background, white);
}

.disabled-div {
    pointer-events: none;
    opacity: 0.7;
}

.md-toolbar.md-theme-default{
  background-color: var(--md-theme-default-background-2, white);
}

.md-title{
  /*--md-theme-default-text-primary-on-background-variant*/
  color: var(--md-theme-default-primary, black) !important;
}

.datepicker-label{
  color: var(--md-theme-default-background-4, black) !important;
}

.md-datepicker-dialog.md-theme-default .md-datepicker-header{
  background-color: var(--md-theme-default-accent-on-background, #448aff);
}

.md-datepicker-dialog.md-theme-default .md-datepicker-day-button.md-datepicker-selected{
  background-color: var(--md-theme-default-accent-on-background, #448aff);
}

.md-datepicker-dialog.md-theme-default .md-datepicker-body-footer{
  /*--md-theme-default-background*/
  background-color: var(--md-theme-default-background-2, white);
}

.md-datepicker-body-content{
  background-color: var(--md-theme-default-background-2, white);
}

.md-datepicker-dialog.md-theme-default .md-datepicker-body-header:before{
  background-color: var(--md-theme-default-background-2, white);
}

.md-datepicker-dialog.md-theme-default .md-datepicker-body-header:after{
  background-color: var(--md-theme-default-background-2, white);
}

.md-datepicker-year-select, .md-datepicker-dayname, .md-datepicker-monthname, .md-datepicker-day {
  color: white;
}

.md-button.md-theme-default.md-raised[disabled]{
  background-color:rgba(150, 150, 150, 0.15);
}

.md-toolbar.md-theme-default .md-icon{
  color: var(--md-theme-default-background-3, black);
  -webkit-text-fill-color: var(--md-theme-default-background-3, black);
}

:root {
    /* --primary-color: #302AE6;
    --secondary-color: #536390;
    --font-color: #424242;
    --bg-color: #fff;
    --heading-color: #292922; */
    --md-theme-default-primary: black;
    --md-theme-default-primary-transparent: rgba(255, 255, 255, 0.85);
    --md-theme-default-background: white;
    --md-theme-default-background-2: white;
    --md-theme-default-background-3: rgba(0, 0, 0, 0.85);
    --md-theme-default-background-4: rgba(0, 0, 0, 0.5);
    --md-theme-default-background-5: rgba(0, 0, 0, 0.20);
    --md-theme-default-accent-on-background: #ff5252;
    --md-theme-default-divider-on-background: rgba(0,0,0,0.12);
}

[data-theme="darkMode"] {
    /* --primary-color: #9A97F3;
    --secondary-color: #818cab;
    --font-color: #e1e1ff;
    --bg-color: #161625;
    --heading-color: #818cab; */
    --md-theme-default-primary: white;
    --md-theme-default-primary-transparent: rgba(0, 0, 0, 0.85);
    --md-theme-default-background: #1d1d1d;
    --md-theme-default-background-2: #272727;
    --md-theme-default-background-3: rgba(255, 255, 255, 0.85);
    --md-theme-default-background-4: rgba(255, 255, 255, 0.5);
    --md-theme-default-background-5: rgba(255, 255, 255, 0.20);
    --md-theme-default-accent-on-background: #ff5252;
    --md-theme-default-divider-on-background: rgba(255,255,255,0.12);
}
</style>
