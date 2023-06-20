<script>
import mqtt from "mqtt";

export default {
  data() {
    return {
      connection: {
        protocol: "ws",
        host: "localhost",
        port: 8083,
        endpoint: null,
        clean: false,
        connectTimeout: 10 * 1000, 
        reconnectPeriod: 4000, 
        clientId: "driver_" + Math.random().toString(16).substring(2, 8),
        // auth
        username: "drivertest",
        password: "123456",
      },

      subscription: {
        topic: "",
        qos: 2,
      },

      publish: {
        topic: this.subscription.topic,
        qos: 0,
        payload: '{ "msg": "Hello, I am browser." }',
      },

      receiveNews: "",
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
          this.$message.error("Connection maxReconnectTimes limit, stop retry");
        } catch (error) {
          this.$message.error(error.toString());
        }
      }
    },
    createConnection() {
      try {
        this.connecting = true;
        const { protocol, host, port, endpoint, ...options } = this.connection;
        const connectUrl = `${protocol}://${host}:${port}${endpoint}`;
        this.client = mqtt.connect(connectUrl, options);
        if (this.client.on) {
          this.client.on("connect", () => {
            this.connecting = false;
            console.log("Connection succeeded!");
          });
          this.client.on("reconnect", this.handleOnReConnect);
          this.client.on("error", (error) => {
            console.log("Connection failed", error);
          });
          this.client.on("message", (topic, message) => {
            this.receiveNews = this.receiveNews.concat(message);
            console.log(`Received message ${message} from topic ${topic}`);
          });
        }
      } catch (error) {
        this.connecting = false;
        console.log("mqtt.connect error", error);
      }
    },

    doSubscribe() {
      const { topic, qos } = this.subscription
      this.client.subscribe(topic, { qos }, (error, res) => {
        if (error) {
          console.log('Subscribe to topics error', error)
          return
        }
        this.subscribeSuccess = true
        console.log('Subscribe to topics res', res)
      })
    },

    doUnSubscribe() {
      const { topic } = this.subscription
      this.client.unsubscribe(topic, error => {
        if (error) {
          console.log('Unsubscribe error', error)
        }
      })
    },

    doPublish() {
      const { topic, qos, payload } = this.publish
      this.client.publish(topic, payload, { qos }, error => {
        if (error) {
          console.log('Publish error', error)
        }
      })
    },

    destroyConnection() {
      if (this.client.connected) {
        try {
          this.client.end(false, () => {
            this.initData()
            console.log('Successfully disconnected!')
          })
        } catch (error) {
          console.log('Disconnect failed', error.toString())
        }
      }
    },

    getChannelFromCoordinates() {
      return "ride-Alabama-Montgomery";
    },
  },

  created() {
    // 在 created 生命周期钩子函数中，根据坐标设置 topic
    this.subscription.topic = this.getChannelFromCoordinates();
  }
};
</script>

<template>
  <div>
    <h1>MQTT Client</h1>
    <button v-if="!connecting && !client.connected" @click="createConnection">Connect</button>
    <button v-else-if="connecting" disabled>Connecting...</button>
    <button v-else-if="client.connected" @click="destroyConnection">Disconnect</button>
    <button v-else disabled>Disconnected</button>

    <h2>Subscription</h2>
    <button v-if="client.connected && !subscribeSuccess" @click="doSubscribe">Subscribe</button>
    <button v-else-if="client.connected && subscribeSuccess" @click="doUnSubscribe">Unsubscribe</button>

    <h2>Publish</h2>
    <button v-if="client.connected && subscribeSuccess" @click="doPublish">Publish Message</button>

    <h2>Received Messages</h2>
    <p>{{ receiveNews }}</p>
  </div>
</template>
