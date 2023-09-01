<template>
  <v-list-item style="padding: 5px; height: auto">
    <ListItem
      class="button-hover-thing"
      :class="{ 'selected-player': store.selectedPlayer?.player_id == player.player_id }"
      @click="store.selectedPlayer = player"
    >
      <!-- Player icon -->
      <template #prepend>
        <v-icon
          size="25"
          style="opacity: 100%"
          :icon="player.group_childs.length > 0 ? 'mdi-speaker-multiple' : 'mdi-speaker'"
          color="#FFF"
        />
      </template>

      <!-- Player name -->
      <template #title>
        <div style="margin-top: 5px">
          <b>{{ truncateString(getPlayerName(player), 20) }}</b>
        </div>
      </template>

      <!-- Currently playing -->
      <template v-if="player.state == PlayerState.PLAYING" #subtitle>
        <div style="height: 67px">
          <div style="line-height: 1em">
            <p>{{ truncateString(playerQueue?.current_item?.name || 'Nothing playing', 30) }}</p>
          </div>
          <div style="line-height: 1em">
            <PlayerVolume
              v-if="player.state == PlayerState.PLAYING"
              class="vc-slider"
              height="25px"
              width="100%"
              color="white"
              :is-powered="player.powered"
              :model-value="player.group_childs.length > 0 ? player.group_volume : player.volume_level"
              @update:model-value="api.playerCommandVolumeSet(player.player_id, $event)"
            />
          </div>
        </div>
      </template>
      <template #append>
        <div style="opacity: 1">
          <div class="text-center" style="padding-right: 5px">
            <Button style="height: 25px !important; opacity: 1" icon @click="setGroupPower(player, !player.powered)">
              <v-icon :size="25" :icon="player.volume_muted ? 'mdi-volume-off' : 'mdi-power'" />
            </Button>
            <div v-if="player.state == PlayerState.PLAYING" style="opacity: 1" class="text-caption">
              {{ player.group_volume }}
            </div>
          </div>
        </div>
      </template>
    </ListItem>
  </v-list-item>
  <v-divider />
</template>

<script setup lang="ts">
import { Player, PlayerState, PlayerType } from '../plugins/api/interfaces';
import { api } from '../plugins/api';
import { truncateString, getPlayerName } from '@/helpers/utils';
import PlayerVolume from '@/layouts/default/PlayerOSD/PlayerVolume.vue';
import ListItem from '@/components/mods/ListItem.vue';
import Button from '@/components/mods/Button.vue';
import { store } from '@/plugins/store';
import { computed } from 'vue';

export interface Props {
  player: Player;
}
const props = defineProps<Props>();

const playerQueue = computed(() => {
  if (props.player) {
    return api.queues[props.player.active_source];
  }
  return undefined;
});

const getVolumePlayers = function (player: Player) {
  const items: Player[] = [];
  if (player.type != PlayerType.GROUP) {
    items.push(player);
  }
  for (const groupChildId of player.group_childs) {
    const volumeChild = api?.players[groupChildId];

    if (volumeChild && volumeChild.available && !items.includes(volumeChild)) {
      items.push(volumeChild);
    }
  }
  items.sort((a, b) => (a.display_name.toUpperCase() > b.display_name.toUpperCase() ? 1 : -1));
  return items;
};
const setGroupPower = function (player: Player, powered: boolean) {
  if (player.type != PlayerType.GROUP && player.group_childs.length > 0) {
    // send power command to all group child players
    for (const childPlayer of getVolumePlayers(player)) {
      // bypass api throttling by sending the command directly
      api.sendCommand('players/cmd/power', {
        player_id: childPlayer.player_id,
        powered,
      });
    }
  }
  // regular power command to single player (or special group player)
  api.playerCommandPower(player.player_id, powered);
};
</script>

<style>
.vc-slider {
  padding-top: 13px;
  margin-inline-start: 0px !important;
  margin-inline-end: 0px !important;
}

.button-hover-thing {
  transition: 0.3s;
}

.button-hover-thing:hover {
  background-color: #2c2c2c;
}

.selected-player {
  background-color: #373737;
}

.v-list-item__prepend > .v-list-item__spacer {
  width: 0 !important;
}

.v-list-item-subtitle {
  height: 40px;
}
</style>
