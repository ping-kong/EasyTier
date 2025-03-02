<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue'

import Stepper from 'primevue/stepper';
import StepperPanel from 'primevue/stepperpanel';

import { useToast } from "primevue/usetoast";

import {
  i18n, loadLocaleFromLocalStorage, NetworkConfig, parseNetworkConfig,
  useNetworkStore, runNetworkInstance, retainNetworkInstance, collectNetworkInfos,
  changeLocale
} from './main';

import Config from './components/Config.vue';
import Status from './components/Status.vue';

import { exit } from '@tauri-apps/api/process';

const visible = ref(false);
const tomlConfig = ref("");

const items = ref([
  {
    label: () => i18n.global.t('show_config'),
    icon: 'pi pi-file-edit',
    command: async () => {
      try {
        const ret = await parseNetworkConfig(networkStore.curNetwork);
        tomlConfig.value = ret;
      } catch (e: any) {
        tomlConfig.value = e;
      }
      visible.value = true;
    }
  },
  {
    label: () => i18n.global.t('del_cur_network'),
    icon: 'pi pi-times',
    command: async () => {
      networkStore.removeNetworkInstance(networkStore.curNetwork.instance_id);
      await retainNetworkInstance(networkStore.networkInstanceIds);
      networkStore.delCurNetwork();
    },
    disabled: () => networkStore.networkList.length <= 1,
  },
])

enum Severity {
  None = "none",
  Success = "success",
  Info = "info",
  Warn = "warn",
  Error = "error",
}

const messageBarSeverity = ref(Severity.None);
const messageBarContent = ref("");

const toast = useToast();

const networkStore = useNetworkStore();

const addNewNetwork = () => {
  networkStore.addNewNetwork();
  networkStore.curNetwork = networkStore.lastNetwork;
}

const networkMenuName = (network: NetworkConfig) => {
  return network.network_name + " (" + network.instance_id + ")";
}

networkStore.$subscribe(async () => {
  networkStore.saveToLocalStroage();
  try {
    await parseNetworkConfig(networkStore.curNetwork);
    messageBarSeverity.value = Severity.None;
  } catch (e: any) {
    messageBarContent.value = e;
    messageBarSeverity.value = Severity.Error;
  }
});

async function runNetworkCb(cfg: NetworkConfig, cb: (e: MouseEvent) => void) {
  cb({} as MouseEvent);
  networkStore.removeNetworkInstance(cfg.instance_id);
  await retainNetworkInstance(networkStore.networkInstanceIds);
  networkStore.addNetworkInstance(cfg.instance_id);

  try {
    await runNetworkInstance(cfg);
  } catch (e: any) {
    console.error(e);
    toast.add({ severity: 'info', detail: e });
  }
}

async function stopNetworkCb(cfg: NetworkConfig, cb: (e: MouseEvent) => void) {
  console.log("stopNetworkCb", cfg, cb);
  cb({} as MouseEvent);
  networkStore.removeNetworkInstance(cfg.instance_id);
  await retainNetworkInstance(networkStore.networkInstanceIds);
}

async function updateNetworkInfos() {
  networkStore.updateWithNetworkInfos(await collectNetworkInfos());
}

let intervalId = 0;
onMounted(() => {
  intervalId = setInterval(async () => {
    await updateNetworkInfos();
  }, 500);
});
onUnmounted(() => clearInterval(intervalId))

const curNetworkHasInstance = computed(() => {
  return networkStore.networkInstanceIds.includes(networkStore.curNetworkId);
});

const activeStep = computed(() => {
  return curNetworkHasInstance.value ? 1 : 0;
});

const setting_menu = ref();
const setting_menu_items = ref([
  {
    label: () => i18n.global.t('settings'),
    items: [
      {
        label: () => i18n.global.t('exchange_language'),
        icon: 'pi pi-refresh',
        command: () => {
          changeLocale((i18n.global.locale.value === 'en' ? 'cn' : 'en'));
        }
      },
      {
        label: () => i18n.global.t('exit'),
        icon: 'pi pi-times',
        command: async () => {
          await exit(1);
        }
      }
    ]
  }
]);

const toggle_setting_menu = (event: any) => {
  setting_menu.value.toggle(event);
};

onMounted(async () => {
  networkStore.loadFromLocalStorage();
  changeLocale(loadLocaleFromLocalStorage());
});

</script>

<template>
  <!-- <n-config-provider :theme="lightTheme"> -->
  <div id="root" class="flex flex-column">
    <Dialog v-model:visible="visible" modal header="Config File" :style="{ width: '70%' }">
      <Panel>
        <ScrollPanel style="width: 100%; height: 300px">
          <pre>{{ tomlConfig }}</pre>
        </ScrollPanel>
      </Panel>
      <Divider />
      <div class="flex justify-content-end gap-2">
        <Button type="button" :label="$t('close')" @click="visible = false"></Button>
      </div>
    </Dialog>

    <div>
      <Toolbar>
        <template #start>
          <div class="flex align-items-center gap-2">
            <Button icon="pi pi-plus" class="mr-2" severity="primary" :label="$t('add_new_network')"
              @click="addNewNetwork" />
          </div>
        </template>

        <template #center>
          <div class="min-w-80 mr-20">
            <Dropdown v-model="networkStore.curNetwork" :options="networkStore.networkList"
              :optionLabel="networkMenuName" :placeholder="$t('select_network')" :highlightOnSelect="true"
              :checkmark="true" class="w-full md:w-32rem" />
          </div>
        </template>

        <template #end>
          <Button icon="pi pi-cog" class="mr-2" severity="secondary" aria-haspopup="true" @click="toggle_setting_menu"
            :label="$t('settings')" aria-controls="overlay_setting_menu" />
          <Menu ref="setting_menu" id="overlay_setting_menu" :model="setting_menu_items" :popup="true" />
        </template>
      </Toolbar>
    </div>

    <Stepper class="h-full overflow-y-auto" :active-step="activeStep">
      <StepperPanel :header="$t('config_network')" class="w">
        <template #content="{ nextCallback }">
          <Config @run-network="runNetworkCb($event, nextCallback)" :instance-id="networkStore.curNetworkId"
            :config-invalid="messageBarSeverity != Severity.None" />
        </template>
      </StepperPanel>
      <StepperPanel :header="$t('running')">
        <template #content="{ prevCallback }">
          <div class="flex flex-column">
            <Status :instance-id="networkStore.curNetworkId" />
          </div>
          <div class="flex pt-4 justify-content-center">
            <Button :label="$t('stop_network')" severity="danger" icon="pi pi-arrow-left"
              @click="stopNetworkCb(networkStore.curNetwork, prevCallback)" />
          </div>
        </template>
      </StepperPanel>
    </Stepper>

    <div>
      <Menubar :model="items" breakpoint="300px">
      </Menubar>
      <InlineMessage v-if="messageBarSeverity !== Severity.None" class="absolute bottom-0 right-0" severity="error">
        {{ messageBarContent }}</InlineMessage>
    </div>
  </div>

</template>

<style scoped>
#root {
  height: 100vh;
  width: 100vw;
}
</style>

<style>
body {
  height: 100vh;
  width: 100vw;
  padding: 0;
  margin: 0;
  overflow: hidden;
}

.p-menubar .p-menuitem {
  margin: 0;
}

/* 

.p-tabview-panel {
  height: 100%;
} */
</style>


<script lang="ts">
</script>