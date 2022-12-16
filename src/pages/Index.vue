<template>
  <div>
    <q-card class="q-pa-lg">
      <span class="text-h6 q-mb-lg block">Configuration</span>
      <q-form class="form-config" @submit="createConnection" @reset="destroyConnection">
        <div class="q-gutter-lg row q-py-md">
          <div class="col-md-3">
            <p>Host</p>
            <q-input outlined dense v-model="connection.host" />
          </div>
          <div class="col-md-3">
            <p>Port</p>
            <q-input outlined dense v-model="connection.port" />
          </div>
          <div class="col-md-3">
            <p>Mountpoint</p>
            <q-input outlined dense v-model="connection.endpoint" />
          </div>
          <div class="col-md-3">
            <p>Client ID</p>
            <q-input outlined dense v-model="connection.clientId" />
          </div>
          <div class="col-md-3">
            <p>Username</p>
            <q-input outlined dense v-model="connection.username" />
          </div>
          <div class="col-md-3">
            <p>Password</p>
            <q-input outlined dense v-model="connection.password" />
          </div>
        </div>
        <div class="q-mt-md">
          <q-btn
            :disable="client.connected"
            :label="client.connected ? 'Connected' : 'Connect'"
            type="submit"
            color="positive"
          />
          <q-btn v-if="client.connected" type="reset" class="q-ml-md" label="Disconnect" color="negative" />
        </div>
      </q-form>
    </q-card>
    <q-card class="q-pa-lg q-mt-xl">
      <span class="text-h6 q-mb-lg block">Subscribe</span>
      <q-form @submit="doSubscribe" @reset="doUnSubscribe">
        <div class="q-gutter-lg row items-end">
          <div class="col-md-3">
            <p>Topic</p>
            <q-input outlined dense v-model="subscription.topic" />
          </div>
          <div class="col-md-3">
            <p>QoS</p>
            <q-select outlined dense v-model="subscription.qos" :options="qosList" />
          </div>
          <div class="col-md-3">
            <q-btn
              :disable="!client.connected"
              :label="subscribeSuccess ? 'Subscribed' : 'Subscribe'"
              type="submit"
              color="positive"
            />
            <q-btn :disable="!client.connected" class="q-ml-md" label="Unsubcribe" type="reset" color="positive" />
          </div>
        </div>
      </q-form>
    </q-card>
    <q-card class="q-pa-lg q-mt-xl">
      <span class="text-h6 q-mb-lg block">Publish</span>
      <q-form @submit="doPublish">
        <div class="q-gutter-lg row items-end">
          <div class="col-md-3">
            <p>Topic</p>
            <q-input outlined dense v-model="publish.topic" />
          </div>
          <div class="col-md-3">
            <p>Payload</p>
            <q-input outlined dense v-model="publish.payload" />
          </div>
          <div class="col-md-3">
            <p>QoS</p>
            <q-select outlined dense v-model="publish.qos" :options="qosList" />
          </div>
        </div>
        <div class="q-mt-md">
          <q-btn :disable="!client.connected" label="Publish" type="submit" color="positive" />
        </div>
      </q-form>
    </q-card>
    <q-card class="q-pa-lg q-mt-xl">
      <span class="text-h6 q-mb-lg block">Receive</span>
      <q-input v-model="receiveNews" filled type="textarea" />
      <span class="text-h6 q-mb-lg block">Me</span>
      <q-input v-model="giveNews" filled type="textarea" />
    </q-card>
    <q-card class="q-pa-lg q-mt-xl">
      <q-chat-message :text="['hey, how are you?', 'pie']" sent />
      <q-chat-message :text="['doing fine, how r you?']" />
      <q-chat-message :text="['gimana?']" :sent="true" />
    </q-card>
  </div>
</template>

<script>
import mqtt from 'mqtt';

export default {
  data() {
    return {
      name: '',
      connection: {
        protocol: 'wss',
        host: process.env.VUE_APP_HOST,
        // ws: 8083; wss: 8084
        port: 9001,
        endpoint: '/mqtt',
        // for more options, please refer to https://github.com/mqttjs/MQTT.js#mqttclientstreambuilder-options
        clean: true,
        connectTimeout: 30 * 1000, // ms
        reconnectPeriod: 4000, // ms
        clientId: `emqx_vue_${Math.random().toString(16).substring(2, 8)}`,
        // auth
        username: process.env.VUE_APP_USERNAME,
        password: process.env.VUE_APP_PASSWORD,
      },
      subscription: {
        topic: 'topic/test',
        qos: 0,
      },
      publish: {
        topic: 'topic/#',
        qos: 0,
        payload: '{ "msg": "Hello, I am browser." }',
      },
      receiveNews: '',
      giveNews: '',
      qosList: [0, 1, 2],
      client: {
        connected: false,
      },
      subscribeSuccess: false,
      connecting: false,
      retryTimes: 0,
    };
  },

  methods: {
    initData() {
      this.client = {
        connected: false,
      };
      this.retryTimes = 0;
      this.connecting = false;
      this.subscribeSuccess = false;
    },
    handleOnReConnect() {
      this.retryTimes += 1;
      if (this.retryTimes > 5) {
        try {
          this.client.end();
          this.initData();
          this.$message.error('Connection maxReconnectTimes limit, stop retry');
        } catch (error) {
          this.$message.error(error.toString());
        }
      }
    },
    createConnection() {
      try {
        this.connecting = true;
        // eslint-disable-next-line object-curly-newline
        const { protocol, host, port, endpoint, ...options } = this.connection;
        const connectUrl = `${protocol}://${host}:${port}${endpoint}`;
        this.client = mqtt.connect(connectUrl, options);
        if (this.client.on) {
          this.client.on('connect', () => {
            this.connecting = false;
            console.log('Connection succeeded!');
          });
          this.client.on('reconnect', this.handleOnReConnect);
          this.client.on('error', (error) => {
            console.log('Connection failed', error);
          });
          this.client.on('message', (topic, message) => {
            // console.log(topic);
            // console.log(message);
            this.receiveNews = this.receiveNews.concat(message);
            console.log(`Received message ${message} from topic ${topic}`);
          });
        }
      } catch (error) {
        this.connecting = false;
        console.log('mqtt.connect error', error);
      }
    },
    doSubscribe() {
      const { topic, qos } = this.subscription;
      this.client.subscribe(topic, { qos }, (error, res) => {
        if (error) {
          console.log('Subscribe to topics error', error);
          return;
        }
        this.subscribeSuccess = true;
        console.log('Subscribe to topics res', res);
      });
    },
    doUnSubscribe() {
      const { topic } = this.subscription;
      this.client.unsubscribe(topic, (error) => {
        if (error) {
          console.log('Unsubscribe error', error);
        }
      });
    },
    doPublish() {
      const { topic, qos, payload } = this.publish;
      this.client.publish(topic, payload, { qos }, (error) => {
        if (error) {
          console.log('Publish error', error);
        } else {
          // console.log(topic);
          // console.log(payload);
          this.giveNews = this.giveNews.concat(payload);
        }
      });
    },
    destroyConnection() {
      if (this.client.connected) {
        try {
          this.client.end(false, () => {
            this.initData();
            console.log('Successfully disconnected!');
          });
        } catch (error) {
          console.log('Disconnect failed', error.toString());
        }
      }
    },
  },
};
</script>

<style lang="scss" scoped></style>
