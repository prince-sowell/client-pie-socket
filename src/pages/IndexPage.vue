<template>
  <q-page class="justify-evenly q-mt-xl">
    <div
      v-if="userInfos"
      class="q-ml-md"
    >
      <div class="text-h6">Username: {{ userInfos.username }}</div>
    </div>
    <div class="row">
      <div class="q-ml-md">
        <q-btn
          color="primary"
          label="User"
          no-caps
          @click="isCreatUser = true"
        />
      </div>
      <div class="q-ml-md">
        <q-btn
          color="primary"
          label="create room"
          no-caps
          @click="isCreatRoom = true"
        />
      </div>
    </div>
    <div
      class="row justify-beetwen"
      style="max-width: 500px"
      :style="userInfos && 'cursor: pointer'"
    >
      <q-card
        class="my-card q-ma-md"
        style="max-height: 50px"
        :style="roomId == room.id ? 'background: #adeaff' : ''"
        v-for="room in rooms"
        :key="room.id"
        @click="userInfos && findChatRooms(room.id)"
      >
        <q-card-section>
          {{ room.name }}
        </q-card-section>
      </q-card>
    </div>
    <div class="q-mt-lg">
      <div
        class="q-pa-md row justify-center q-ma-sm message_area"
        v-if="roomId && userInfos"
      >
        <div style="width: 100%; max-width: 400px">
          <q-chat-message
            v-for="chat in chatRoom"
            :key="chat.id"
            :name="chat.user_id === userInfos.id ? 'me' : 'other user'"
            :text="[chat.message]"
            :sent="chat.user_id === userInfos.id"
            :stamp="chat.type ? `event: ${chat?.type}` : ''"
          />
        </div>
      </div>
    </div>

    <div class="row">
      <div
        class="row justify-center"
        v-if="roomId"
      >
        <div class="q-ml-md q-mt-sm">
          <q-input
            v-model="message"
            outlined
            placeholder="new message for talk"
            dense
          >
            <template v-slot:append>
              <q-btn
                round
                dense
                flat
                :disable="!message"
                @click="sendMessage"
                :loading="onSend && message.length > 0"
                icon="send"
              />
            </template>
          </q-input>
        </div>
      </div>
      <div
        class="row justify-center"
        v-if="roomId"
      >
        <div class="q-ml-md q-mt-sm">
          <q-input
            v-model="report"
            outlined
            placeholder="new message for report"
            dense
          >
            <template v-slot:append>
              <q-btn
                round
                dense
                flat
                :disable="!report"
                @click="sendMessage"
                :loading="onSend && report.length > 0"
                icon="send"
              />
            </template>
          </q-input>
        </div>
      </div>
      <div
        class="row justify-center"
        v-if="roomId"
      >
        <div class="q-ml-md q-mt-sm">
          <q-input
            v-model="issue"
            outlined
            placeholder="new message for issue"
            dense
          >
            <template v-slot:append>
              <q-btn
                round
                dense
                flat
                :disable="!issue"
                @click="sendMessage"
                :loading="onSend && issue.length > 0"
                icon="send"
              />
            </template>
          </q-input>
        </div>
      </div>
    </div>

    <!-- Dialog -->
    <q-dialog
      v-model="isCreatUser"
      persistent
    >
      <Add_name
        :title="'Add name'"
        @add-name="addName"
      />
    </q-dialog>
    <q-dialog
      v-model="isCreatRoom"
      persistent
    >
      <Add_name
        :title="'Add Name room'"
        @add-name="createRoom"
      />
    </q-dialog>
  </q-page>
</template>

<script lang="ts">

import { defineComponent, ref, onMounted, watch } from 'vue';
import Add_name from 'components/dialogs/Add_name.vue';
import { api } from 'boot/axios';
import { useQuasar } from 'quasar';
import { Chat, Room, User } from 'src/components/models';
// const PieSocket = require('piesocket-js')
import PieSocket from 'piesocket-js';

export default defineComponent({
  name: 'IndexPage',
  components: { Add_name },
  setup() {
    const $q = useQuasar();
    const isCreatUser = ref(false);
    const isCreatRoom = ref(false);
    const userInfos = ref<User>();
    const roomId = ref();
    const message = ref('');
    const issue = ref('');
    const report = ref('');
    const rooms = ref<Room[]>([]);
    const chatRoom = ref<Chat[]>([]);
    const onSend = ref(false)

    // PieSocket
    let piesocket = new PieSocket({
      clusterId: 'demo',
      apiKey: 'VCXCEuvhGcBDP7XhiJJUDvR1e1D3eiVjgZ9VRiaV'
    });

    const addName = async (name: string) => {
      api
        .get(`/user-one/${name}`)
        .then(({ data }) => {
          if (data.message === 'user not found') {
            creatUser(name);
          } else {
            userInfos.value = data;
          }
        })
        .catch(({ response: { data } }) => {
          if (data.errors) {
            $q.notify({
              message: 'probleme sur le serveur',
              icon: 'infos',
              color: 'info',
            });
          }
        });
    };

    const sendMessage = () => {
      if (userInfos.value || roomId.value) {
        let type = ''
        if (message.value) {
          type = 'talks'
        } if (issue.value) {
          type = 'issue'
        }
        if (report.value) {
          type = 'report'
        }

        onSend.value = true
        api
          .post(`rooms/${roomId.value}/room_messages`, {
            'user_id': userInfos.value?.id,
            'room_id': roomId.value,
            'message': message.value || issue.value || report.value,
            'type': type
          })
          .then(() => {
            resetInputs()
            onSend.value = false
          })
          .catch(({ response: { data } }) => {
            if (data.errors) {
              $q.notify({
                message: 'Erreur du serveur',
                icon: 'infos',
                color: 'info',
              });
            }
            onSend.value = false
            resetInputs()
          });

      } else {
        $q.notify({
          message: 'information manquante',
          icon: 'infos',
          color: 'info',
        });
      }
    };

    const resetInputs = () => {
      message.value = ''
      issue.value = ''
      report.value = ''
    }

    const findRooms = () => {
      api
        .get('/rooms')
        .then(({ data }) => {
          rooms.value = data;
        })
        .catch(({ response: { data } }) => {
          if (data.errors) {
            $q.notify({
              message: 'probleme sur le serveur',
              icon: 'infos',
              color: 'info',
            });
          }
        });
    };

    const findChatRooms = (id: number) => {
      roomId.value = id;
      api
        .get(`/rooms/${id}/room_messages`)
        .then(({ data }) => {
          chatRoom.value = [];
          chatRoom.value = data;
        })
        .catch(({ response: { data } }) => {
          if (data.errors) {
            $q.notify({
              message: 'probleme sur le serveur',
              icon: 'infos',
              color: 'info',
            });
          }
        });
    };

    const creatUser = (username: string) => {
      api
        .post('/user', {
          username: username,
        })
        .then(({ data }) => {
          userInfos.value = data;
        })
        .catch(({ response: { data } }) => {
          if (data.errors === 'Username has already been taken') {
            $q.notify({
              message: 'Nom déjà utiliser',
              icon: 'infos',
              color: 'info',
            });
          }
        });
    };

    const createRoom = (username: string) => {
      void api
        .post('/rooms', {
          name: username,
        })
        .then(({ data }) => {
          rooms.value.push(data);
        })
        .catch(({ response: { data } }) => {
          if (data.errors === 'Username has already been taken') {
            $q.notify({
              message: 'Nom déjà utiliser',
              icon: 'infos',
              color: 'info',
            });
          }
        });
    };

    const subscribe = async (channelId: number) => {
      let channel = await piesocket.subscribe(channelId);
      channel.listen('issue', ({ message }: { message: Chat }) => {
        if (roomId.value === message.room_id) chatRoom.value.push({ ...message, type: 'issue' })
      });
      channel.listen('report', ({ message }: { message: Chat }) => {
        if (roomId.value === message.room_id) chatRoom.value.push({ ...message, type: 'report' })
      });
      channel.listen('talks', ({ message }: { message: Chat }) => {
        if (roomId.value === message.room_id) chatRoom.value.push({ ...message, type: 'talks' })
      });
    }

    watch(roomId, (currentValue) => {
      subscribe(currentValue)
    });

    onMounted(async () => {
      isCreatUser.value = true;
      findRooms();
    });

    return {
      isCreatUser,
      roomId,
      rooms,
      userInfos,
      isCreatRoom,
      chatRoom,
      message,
      onSend,
      issue,
      report,
      addName,
      sendMessage,
      createRoom,
      findChatRooms,
    };
  },
});
</script>
<style>
.message_area {
  border: 1px solid #c9c9c9;
  border-radius: 5px
}
</style>