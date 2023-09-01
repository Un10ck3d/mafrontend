<template>
  <!-- players menu -->
  <v-card
    v-if="store.showPlayersMenu"
    v-click-outside="{
      handler: () => {
        store.showPlayersMenu = !store.showPlayersMenu;
      },
      include: excludeClick,
    }"
    width="280"
    location="bottom end"
    transition="scale-transition"
    style="z-index: 9999; position: absolute"
  >
    <!-- heading with Players as title-->
    <v-card-title style="padding: 10px; justify-content: space-between; display: flex" class="headline">
      <h1 style="margin-top: auto; margin-bottom: auto; margin-left: 5px">{{ $t('players') }}</h1>
      <!-- settings button -->
      <div>
        <v-btn
          variant="plain"
          style="font-size: large; font-weight: bolder; opacity: 100%"
          icon="mdi-cog-outline"
          to="/settings/players"
          @click="store.showPlayersMenu = !store.showPlayersMenu"
        />
        <!-- close button in the top right (accessibility reasons)-->
        <v-btn
          style="font-size: x-large; font-weight: bolder; opacity: 100%"
          variant="plain"
          icon="mdi-close"
          @click="store.showPlayersMenu = !store.showPlayersMenu"
        />
      </div>
    </v-card-title>

    <v-divider />

    <v-expansion-panels v-model="panelItem" mandatory="force" focusable accordion flat>
      <v-expansion-panel>
        <v-expansion-panel-title style="padding: 15px"
          ><h3 style="font-weight: bold">Currently Playing</h3></v-expansion-panel-title
        >
        <v-expansion-panel-text variant="contain">
          <v-list flat>
            <PlayerControl v-for="player in activePlayers" :key="player.player_id" :player="player" />
          </v-list>
        </v-expansion-panel-text>
      </v-expansion-panel>
      <v-expansion-panel>
        <v-expansion-panel-title style="padding: 15px"
          ><h3 style="font-weight: bold">Other Players</h3></v-expansion-panel-title
        >
        <v-expansion-panel-text variant="contain">
          <v-list flat>
            <PlayerControl v-for="player in nonActivePlayers" :key="player.player_id" :player="player" />
          </v-list>
        </v-expansion-panel-text>
      </v-expansion-panel>
      <v-expansion-panel>
        <v-expansion-panel-title style="padding: 15px"
          ><h3 style="font-weight: bold">Other Groups</h3></v-expansion-panel-title
        >
        <v-expansion-panel-text variant="contain">
          <v-list flat>
            <PlayerControl v-for="player in groupedPlayers" :key="player.player_id" :player="player" />
          </v-list>
        </v-expansion-panel-text>
      </v-expansion-panel>
    </v-expansion-panels>
  </v-card>
</template>

<script setup lang="ts">
import { computed, getCurrentInstance, onMounted, ref, watch } from 'vue';
import { Player, PlayerState } from '@/plugins/api/interfaces';
import { store } from '@/plugins/store';
import PlayerControl from '@/components/PlayerControl.vue';
import { api } from '@/plugins/api';
import { getPlayerName, truncateString } from '@/helpers/utils';
import ListItem from '@/components/mods/ListItem.vue';

const panelItem = ref<number | undefined>(undefined);

// computed properties
const sortedPlayers = computed(() => {
  const res: Player[] = [];
  for (const player_id in api?.players) {
    const player = api?.players[player_id];
    // ignore disabled/hidden/synced players
    if (!playerActive(player, true)) continue;
    res.push(player);
  }
  return res.slice().sort((a, b) => (a.display_name.toUpperCase() > b.display_name?.toUpperCase() ? 1 : -1));
});

const activePlayers = computed(() => {
  const res: Player[] = [];
  for (const player of sortedPlayers.value) {
    // ignore non playing players
    if (player.state != PlayerState.PLAYING) continue;
    // ignore groups
    if (player.group_childs.length > 0) continue;
    res.push(player);
  }
  return res;
});

const nonActivePlayers = computed(() => {
  const res: Player[] = [];
  for (const player of sortedPlayers.value) {
    // ignore playing players
    if (player.state == PlayerState.PLAYING) continue;
    // ignore groups
    if (player.group_childs.length > 0) continue;
    res.push(player);
  }
  return res;
});

const groupedPlayers = computed(() => {
  const res: Player[] = [];
  for (const player of sortedPlayers.value) {
    // ignore playing players
    if (player.state == PlayerState.PLAYING) continue;
    // ignore non groups
    if (player.group_childs.length == 0) continue;
    res.push(player);
  }
  return res;
});

//watchers
watch(
  () => store.showPlayersMenu,
  (newVal) => {
    if (!newVal) {
      panelItem.value = undefined;
    }
  },
);
watch(
  () => store.selectedPlayer,
  (newVal) => {
    if (newVal) {
      // remember last selected playerId
      localStorage.setItem('mass.LastPlayerId', newVal.player_id);
    }
  },
);
watch(
  () => api.players,
  (newVal) => {
    if (newVal) {
      checkDefaultPlayer();
    }
  },
  { deep: true },
);

const shadowRoot = ref<ShadowRoot>();
const lastClicked = ref();
onMounted(() => {
  shadowRoot.value = getCurrentInstance()?.vnode?.el?.getRootNode();
  checkDefaultPlayer();
});

const excludeClick = () => {
  return [document.querySelector('.excludeClosePlayerSelector')];
};

const playerActive = function (
  player: Player,
  allowUnavailable = true,
  allowSyncChild = false,
  allowHidden = false,
): boolean {
  // perform some basic checks if we may use/show the player
  if (!player.enabled) return false;
  if (!player.available && !allowUnavailable) return false;
  if (player.synced_to && !allowSyncChild) return false;
  if (player.hidden_by.length && !allowHidden) return false;
  return true;
};

const checkDefaultPlayer = function () {
  if (store.selectedPlayer && playerActive(store.selectedPlayer, false, false, false)) return;
  const newDefaultPlayer = selectDefaultPlayer();
  if (newDefaultPlayer) {
    store.selectedPlayer = newDefaultPlayer;
    console.log('Selected new default player: ', newDefaultPlayer.display_name);
  }
};

const selectDefaultPlayer = function () {
  // check if we have a player stored that was last used
  const lastPlayerId = localStorage.getItem('mass.LastPlayerId');
  if (lastPlayerId) {
    if (lastPlayerId in api.players && playerActive(api.players[lastPlayerId], false, false, false)) {
      return api.players[lastPlayerId];
    }
  }
  // select a (new) default active player
  if (api?.players) {
    // prefer the first playing player
    for (const playerId in api?.players) {
      const player = api.players[playerId];
      if (player.state == PlayerState.PLAYING) {
        return player;
      }
    }
    // fallback to just a player with item in queue
    for (const queueId in api?.queues) {
      const queue = api.queues[queueId];
      if (!playerActive(api.players[queueId], false, false, false)) continue;
      if (queue.items) {
        return api.players[queueId];
      }
    }
    // last resort: just the first queue
    for (const playerId in api?.queues) {
      if (!playerActive(api.players[playerId], false, false, false)) continue;
      return api.players[playerId];
    }
  }
};
</script>

<style>
.playerrow {
  height: 60px;
  margin-right: 15px;
}

div.v-expansion-panel-text__wrapper {
  padding-left: 0px;
  padding-right: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
}

div.v-expansion-panel--active:not(:first-child),
.v-expansion-panel--active + .v-expansion-panel {
  margin-top: 0px;
}

div.v-expansion-panel__shadow {
  box-shadow: none;
}

.v-expansion-panel-title__icon {
  display: inline-flex;
  margin-bottom: -4px;
  margin-top: -4px;
  user-select: none;
  margin-inline-start: auto;
  margin-right: 5px;
}
</style>
