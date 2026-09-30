<template>
  <div>
    <h2 class="mb-4">{{ $t('plugin_kubernetes.title') }}</h2>

    <v-skeleton-loader v-if="loading" :loading="true" type="card" />

    <div v-else style="margin-bottom: 80px">
      <!-- Status Card -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-kubernetes</v-icon>
          <span>{{ $t('plugin_kubernetes.status') }}</span>
        </v-card-title>
        <v-card-text>
          <v-row>
            <v-col cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_kubernetes.version') }} {{ currentVersion || $t('plugin_kubernetes.not_installed') }}
              </span>
            </v-col>
            <v-col cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_kubernetes.latest_version') }} {{ latestVersion || '-' }}
              </span>
            </v-col>
            <v-col cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_kubernetes.status') }}: {{ running ? $t('plugin_kubernetes.running') : $t('plugin_kubernetes.not_running') }}
              </span>
            </v-col>
            <v-col cols="12" md="auto" class="d-flex ga-2 flex-nowrap flex-md-row">
              <v-btn size="small" variant="tonal" color="secondary" @click="checkForUpdates" :loading="checkingUpdates">
                {{ $t('plugin_kubernetes.check_updates') }}
              </v-btn>
              <v-btn
                v-if="!currentVersion && dirSaved && settings.directory"
                size="small" variant="tonal" color="primary"
                @click="doInstall" :loading="installing"
              >
                {{ $t('plugin_kubernetes.install') }}
              </v-btn>
              <v-btn
                v-else-if="currentVersion && updateAvailable"
                size="small" variant="tonal" color="primary"
                @click="doUpdate" :loading="updating"
              >
                {{ $t('plugin_kubernetes.update') }}
              </v-btn>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>

      <!-- Settings Card -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-harddisk</v-icon>
          <span>{{ $t('plugin_kubernetes.settings') }}</span>
        </v-card-title>
        <v-card-text>
          <v-text-field
            v-model="settings.directory"
            :label="$t('plugin_kubernetes.directory')"
            :disabled="running"
            :hint="running ? $t('plugin_kubernetes.directory_locked') : ''"
            :persistent-hint="running"
          >
            <template #append-inner>
              <v-btn size="small" icon="mdi-folder" @click="openFsDialog" variant="text" :disabled="running" />
            </template>
          </v-text-field>
          <div v-if="!dirSaved" class="text-subtitle-2 text-warning mt-2">
            {{ $t('plugin_kubernetes.no_path_set') }}{{ recommendedPath ? ' ' + recommendedPath : '' }}
          </div>
          <template v-if="dirSaved">
            <v-divider class="my-4" />
            <v-select
              v-model="settings.mode"
              :items="nodeModeOptions"
              item-title="title"
              item-value="value"
              :label="$t('plugin_kubernetes.mode')"
              :disabled="running"
              class="mb-2"
            />
            <v-text-field
              v-model="settings.start_args"
              :label="$t('plugin_kubernetes.start_args')"
              :hint="running ? $t('plugin_kubernetes.start_args_locked') : (settings.mode === 'agent' ? $t('plugin_kubernetes.start_args_hint_agent') : $t('plugin_kubernetes.start_args_hint_server'))"
              persistent-hint
              :disabled="running"
            />
            <v-alert
              v-if="!running && settings.mode === 'server'"
              type="warning"
              variant="tonal"
              density="compact"
              class="mt-3"
              :text="$t('plugin_kubernetes.start_args_warning')"
            />
            <v-switch
              v-model="settings.auto_start"
              :label="$t('plugin_kubernetes.auto_start')"
              inset color="green" hide-details
              class="mt-2"
            />
            <v-switch
              v-model="settings.auto_update"
              :label="$t('plugin_kubernetes.auto_update')"
              inset color="green" hide-details
              class="mt-2"
            />
          </template>
          <v-btn color="primary" class="mt-4" @click="confirmSaveSettings" :loading="saving">
            <v-icon start>mdi-content-save</v-icon>
            {{ $t('plugin_kubernetes.save_settings') }}
          </v-btn>
        </v-card-text>
      </v-card>

      <!-- Start/Stop Card -->
      <v-card v-if="dirSaved && currentVersion" class="mb-4 pa-0">
        <v-card-text class="d-flex align-center ga-2">
          <v-btn color="primary" rounded :loading="starting" @click="startDaemon">
            <v-icon start>mdi-play</v-icon>
            {{ $t('plugin_kubernetes.start') }}
          </v-btn>
          <v-btn color="error" rounded variant="outlined" :loading="stopping" @click="stopDaemon">
            <v-icon start>mdi-stop</v-icon>
            {{ $t('plugin_kubernetes.stop') }}
          </v-btn>
        </v-card-text>
      </v-card>

      <!-- Live Log Viewer -->
      <v-expansion-panels v-if="dirSaved && currentVersion" v-model="logPanel" class="mb-4">
        <v-expansion-panel>
          <v-expansion-panel-title>
            <v-icon class="mr-2">mdi-text-box-outline</v-icon>
            {{ $t('plugin_kubernetes.logs') }}
          </v-expansion-panel-title>
          <v-expansion-panel-text>
            <div
              ref="logContainer"
              @scroll="onLogScroll"
              style="height: 720px; overflow-y: auto; white-space: pre-wrap; word-break: break-word; font-family: monospace; font-size: 12px; line-height: 1.4; background: #1a1a1a; color: #e8e8e8; padding: 12px; border-radius: 4px;"
            >{{ logContent || $t('plugin_kubernetes.logs_empty') }}</div>
          </v-expansion-panel-text>
        </v-expansion-panel>
      </v-expansion-panels>
    </div>

    <!-- Restart Confirmation Dialog -->
    <v-dialog v-model="restartDialog" max-width="400">
      <v-card :title="$t('plugin_kubernetes.restart_required')" prepend-icon="mdi-restart" class="pa-0">
        <v-card-text>
          {{ $t('plugin_kubernetes.restart_confirm_message') }}
        </v-card-text>
        <v-divider />
        <v-card-actions>
          <v-row class="d-flex justify-end">
            <v-btn color="onPrimary" @click="restartDialog = false">{{ $t('plugin_kubernetes.cancel') }}</v-btn>
            <v-btn color="primary" @click="restartDialog = false; saveSettings(true)">{{ $t('plugin_kubernetes.ok') }}</v-btn>
          </v-row>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- File System Dialog -->
    <v-dialog v-model="fsDialog" max-width="800">
      <v-card>
        <v-card-title class="d-flex align-center">
          <span>{{ $t('plugin_kubernetes.select_directory') }}</span>
          <v-spacer />
          <v-chip size="small" class="ml-2" variant="tonal">{{ fsCurrentPath || '/' }}</v-chip>
        </v-card-title>
        <v-card-subtitle class="pb-0">
          <div class="d-flex align-center ga-2">
            <v-btn size="small" variant="text" icon="mdi-home" @click="fsGoRoot" color="secondary" :disabled="fsLoading" />
            <v-btn size="small" variant="text" icon="mdi-arrow-up" @click="fsNavigateUp" color="secondary" :disabled="fsCurrentPath === '/mnt' || fsLoading" />
            <v-btn size="small" variant="text" icon="mdi-refresh" @click="fetchDirectory(fsCurrentPath)" color="secondary" :disabled="fsLoading" />
            <v-spacer />
            <v-progress-circular v-if="fsLoading" indeterminate size="20" color="secondary" />
          </div>
        </v-card-subtitle>
        <v-card-text class="pt-2" style="min-height: 300px; max-height: 60vh; overflow-y: auto">
          <v-table density="compact">
            <thead>
              <tr>
                <th>{{ $t('plugin_kubernetes.name') }}</th>
                <th style="width: 40%">{{ $t('plugin_kubernetes.path') }}</th>
                <th style="width: 60px" class="text-center">{{ $t('plugin_kubernetes.action') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="!fsLoading && fsItems.length === 0">
                <td colspan="3" class="text-center text-medium-emphasis">{{ $t('plugin_kubernetes.no_entries') }}</td>
              </tr>
              <tr
                v-for="item in fsItems"
                :key="item.path"
                :class="['cursor-pointer', fsActiveItem && fsActiveItem.path === item.path ? 'bg-primary bg-opacity-10' : '']"
                @click="fsActiveItem = item"
                @dblclick.stop.prevent="fsNavigateInto(item)"
              >
                <td>
                  <div class="d-flex align-center ga-2">
                    <v-icon size="18">mdi-folder</v-icon>
                    <span>{{ item.name }}</span>
                  </div>
                </td>
                <td><span class="text-caption">{{ item.displayPath || item.path }}</span></td>
                <td class="text-center">
                  <v-btn size="small" icon="mdi-folder-open" variant="text" @click.stop="fsNavigateInto(item)" :disabled="fsLoading" />
                </td>
              </tr>
            </tbody>
          </v-table>
        </v-card-text>
        <v-divider />
        <v-card-actions class="d-flex align-center">
          <div class="text-caption text-truncate" style="max-width: 60%">
            <strong>{{ $t('plugin_kubernetes.selected') }}:</strong>
            <span v-if="fsActiveItem">{{ fsActiveItem.displayPath || fsActiveItem.path }}</span>
            <span v-else>-</span>
          </div>
          <v-spacer />
          <v-btn variant="text" color="onPrimary" @click="fsDialog = false">{{ $t('plugin_kubernetes.cancel') }}</v-btn>
          <v-btn color="onPrimary" @click="fsSelect" :disabled="fsLoading">{{ $t('plugin_kubernetes.select') }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, watch, nextTick, getCurrentInstance, onMounted, onUnmounted } from 'vue';

const PLUGIN_NAME = 'kubernetes';

const t = (key) => getCurrentInstance()?.appContext.config.globalProperties.$t(key) ?? key;

const loading = ref(true);
const saving = ref(false);
const starting = ref(false);
const stopping = ref(false);
const checkingUpdates = ref(false);
const updating = ref(false);
const installing = ref(false);
const fsDialog = ref(false);
const fsCurrentPath = ref('');
const fsItems = ref([]);
const fsLoading = ref(false);
const fsActiveItem = ref(null);
const statusInterval = ref(null);

const running = ref(false);
const currentVersion = ref('');
const latestVersion = ref('');
const updateAvailable = ref(false);

const logPanel = ref(undefined);
const logContent = ref('');
const logContainer = ref(null);
const logInterval = ref(null);
const autoScrollLog = ref(true);

const DEFAULT_START_ARGS = '--disable=traefik --disable=servicelb';

const nodeModeOptions = [
  { title: t('plugin_kubernetes.mode_server'), value: 'server' },
  { title: t('plugin_kubernetes.mode_agent'), value: 'agent' },
];

const settings = reactive({
  directory: '',
  mode: 'server',
  auto_start: false,
  auto_update: false,
  start_args: DEFAULT_START_ARGS,
  version: ''
});

const dirSaved = ref(false);
const recommendedPath = ref('');
const restartDialog = ref(false);
const savedDirectory = ref('');
const savedStartArgs = ref(DEFAULT_START_ARGS);
const savedMode = ref('server');

watch(() => settings.mode, (mode, previousMode) => {
  if (mode === 'agent' && previousMode === 'server' && settings.start_args === DEFAULT_START_ARGS) {
    settings.start_args = '';
  } else if (mode === 'server' && previousMode === 'agent' && !settings.start_args.trim()) {
    settings.start_args = DEFAULT_START_ARGS;
  }
});

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const scrollLogToBottom = () => {
  const el = logContainer.value;
  if (el) el.scrollTop = el.scrollHeight;
};

// Keep auto-scrolling only while the user is near the bottom
const onLogScroll = () => {
  const el = logContainer.value;
  if (!el) return;
  autoScrollLog.value = (el.scrollHeight - el.scrollTop - el.clientHeight) < 40;
};

const fetchLogs = async () => {
  try {
    const res = await queryK3s(['logs'], 10, false);
    if (res.ok) {
      const data = await res.json();
      logContent.value = typeof data.output === 'string' ? data.output : '';
      if (autoScrollLog.value) {
        await nextTick();
        scrollLogToBottom();
      }
    }
  } catch (e) {
    // ignore transient log fetch errors
  }
};

watch(logPanel, async (val) => {
  if (val === 0) {
    autoScrollLog.value = true;
    await fetchLogs();
    await nextTick();
    scrollLogToBottom();
    if (!logInterval.value) {
      logInterval.value = setInterval(fetchLogs, 2000);
    }
  } else if (logInterval.value) {
    clearInterval(logInterval.value);
    logInterval.value = null;
  }
});

// Derives the recommended k3s path from the Docker system directory
// e.g. /mnt/pool/system/docker -> /mnt/pool/system/k3s
const deriveK3sPath = (dockerDir) => {
  if (!dockerDir) return '';
  if (/\/system\/docker\/?$/.test(dockerDir)) {
    return dockerDir.replace(/\/system\/docker\/?$/, '/system/k3s');
  }
  if (/\/docker\/?$/.test(dockerDir)) {
    return dockerDir.replace(/\/docker\/?$/, '/k3s');
  }
  return dockerDir.replace(/\/$/, '') + '/k3s';
};

const queryK3s = async (args, timeout = 30, parse_json = false) => {
  return fetch('/api/v1/mos/plugins/query', {
    method: 'POST',
    headers: {
      ...getAuthHeaders(),
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      command: 'k3s',
      args,
      timeout,
      parse_json,
    }),
  });
};

const fetchSettings = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.directory !== undefined && data.directory) {
        settings.directory = data.directory;
        savedDirectory.value = data.directory;
        dirSaved.value = true;
        recommendedPath.value = '';
      } else {
        settings.directory = '';
        savedDirectory.value = '';
        dirSaved.value = false;
      }
      settings.mode = data.mode === 'agent' ? 'agent' : 'server';
      savedMode.value = settings.mode;
      if (data.auto_start !== undefined) {
        settings.auto_start = data.auto_start;
      }
      if (data.auto_update !== undefined) {
        settings.auto_update = data.auto_update;
      }
      if (data.start_args !== undefined && data.start_args !== null) {
        settings.start_args = data.start_args;
      } else {
        settings.start_args = DEFAULT_START_ARGS;
      }
      savedStartArgs.value = settings.start_args;
      if (data.version) {
        currentVersion.value = data.version;
      }
    }

    if (!dirSaved.value && !settings.directory) {
      try {
        const dockerRes = await fetch('/api/v1/mos/settings/docker', {
          headers: getAuthHeaders(),
        });
        if (dockerRes.ok) {
          const dockerData = await dockerRes.json();
          if (dockerData.directory) {
            recommendedPath.value = deriveK3sPath(dockerData.directory);
            settings.directory = recommendedPath.value;
          }
        }
      } catch (e) {
        console.error('Failed to fetch docker settings:', e);
      }
    }
  } catch (e) {
    console.error('Failed to fetch settings:', e);
  }
};

const saveSettings = async (restart = false) => {
  saving.value = true;
  try {
    await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        directory: settings.directory,
        mode: settings.mode,
        auto_start: settings.auto_start,
        auto_update: settings.auto_update,
        start_args: settings.start_args,
        version: currentVersion.value,
      }),
    });
    dirSaved.value = true;
    recommendedPath.value = '';
    savedDirectory.value = settings.directory;
    savedStartArgs.value = settings.start_args;
    savedMode.value = settings.mode;

    if (restart && running.value) {
      await queryK3s(['restart'], 60);
      await checkStatus();
    }
  } catch (e) {
    console.error('Failed to save settings:', e);
  } finally {
    saving.value = false;
  }
};

const confirmSaveSettings = () => {
  const restartNeeded = running.value &&
    (settings.directory !== savedDirectory.value ||
     settings.start_args !== savedStartArgs.value ||
     settings.mode !== savedMode.value);

  if (restartNeeded) {
    restartDialog.value = true;
    return;
  }
  saveSettings(false);
};

const doInstall = async () => {
  installing.value = true;
  try {
    if (!settings.directory) {
      alert('Error: Directory not set. Please set the directory and save first.');
      installing.value = false;
      return;
    }

    await saveSettings(false);

    const res = await queryK3s(['install_binary'], 120);
    if (!res.ok) {
      throw new Error('Install failed');
    }

    await fetchSettings();
    await checkStatus();
    await checkForUpdates();
  } catch (e) {
    console.error('Failed to install:', e);
    alert('Failed to install k3s. Check logs for details.');
  } finally {
    installing.value = false;
  }
};

const doUpdate = async () => {
  updating.value = true;
  try {
    const wasRunning = running.value;
    if (wasRunning) {
      await queryK3s(['stop'], 30);
    }

    const res = await queryK3s(['install_binary'], 120);
    if (!res.ok) {
      throw new Error('Update failed');
    }

    // Restart the service with the new binary if it was running before
    if (wasRunning) {
      await queryK3s(['start'], 60);
    }

    await fetchSettings();
    await checkStatus();
    await checkForUpdates();
  } catch (e) {
    console.error('Failed to update:', e);
    alert('Failed to update k3s. Check logs for details.');
  } finally {
    updating.value = false;
  }
};

const checkStatus = async () => {
  try {
    const res = await queryK3s(['status'], 5, true);
    if (res.ok) {
      const data = await res.json();
      if (data.success && data.output) {
        running.value = data.output.running === true;
      }
    }
  } catch (e) {
    running.value = false;
  }
};

const checkForUpdates = async () => {
  checkingUpdates.value = true;
  try {
    const res = await queryK3s(['check_version'], 15, true);
    if (res.ok) {
      const data = await res.json();
      if (data.success && data.output) {
        latestVersion.value = data.output.latest || '';
        updateAvailable.value = currentVersion.value && data.output.update_available === true;
      }
    }
  } catch (e) {
    console.error('Failed to check updates:', e);
  } finally {
    checkingUpdates.value = false;
  }
};

const startDaemon = async () => {
  starting.value = true;
  try {
    await queryK3s(['start'], 60);
    await checkStatus();
  } catch (e) {
    console.error('Failed to start:', e);
  } finally {
    starting.value = false;
  }
};

const stopDaemon = async () => {
  stopping.value = true;
  try {
    await queryK3s(['stop'], 30);
    await checkStatus();
  } catch (e) {
    console.error('Failed to stop:', e);
  } finally {
    stopping.value = false;
  }
};

const openFsDialog = async () => {
  fsCurrentPath.value = settings.directory || '/mnt';
  fsDialog.value = true;
  await fetchDirectory(fsCurrentPath.value);
};

const fetchDirectory = async (dirPath) => {
  fsLoading.value = true;
  fsActiveItem.value = null;
  try {
    const url = new URL('/api/v1/mos/fsnavigator', window.location.origin);
    if (dirPath && dirPath !== '/') {
      url.searchParams.set('path', dirPath);
    }
    url.searchParams.set('type', 'directories');
    url.searchParams.set('roots', '/mnt');

    const res = await fetch(url.toString(), {
      headers: getAuthHeaders(),
    });

    if (res.ok) {
      const data = await res.json();
      fsCurrentPath.value = data.currentPath || dirPath || '/mnt';
      fsItems.value = Array.isArray(data.items) ? data.items : [];
    } else {
      fsItems.value = [];
    }
  } catch (e) {
    console.error('Failed to fetch directory:', e);
    fsItems.value = [];
  } finally {
    fsLoading.value = false;
  }
};

const fsNavigateInto = (item) => {
  if (!item || item.type !== 'directory') return;
  fetchDirectory(item.path);
};

const fsNavigateUp = () => {
  if (!fsCurrentPath.value || fsCurrentPath.value === '/mnt') return;
  const parts = fsCurrentPath.value.split('/').filter(Boolean);
  parts.pop();
  const parentPath = '/' + parts.join('/');
  fetchDirectory(parentPath || '/mnt');
};

const fsGoRoot = () => {
  fetchDirectory('/mnt');
};

const fsSelect = () => {
  if (fsActiveItem.value) {
    settings.directory = fsActiveItem.value.path;
  } else {
    settings.directory = fsCurrentPath.value;
  }
  fsDialog.value = false;
};

onMounted(async () => {
  try {
    await fetchSettings();
    await checkStatus();
    if (!currentVersion.value) {
      checkForUpdates();
    }

    statusInterval.value = setInterval(async () => {
      await checkStatus();
    }, 5000);
  } catch (e) {
    console.error('Failed to initialize:', e);
  } finally {
    loading.value = false;
  }
});

onUnmounted(() => {
  if (statusInterval.value) {
    clearInterval(statusInterval.value);
  }
  if (logInterval.value) {
    clearInterval(logInterval.value);
  }
});
</script>
