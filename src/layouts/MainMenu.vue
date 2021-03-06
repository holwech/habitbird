<template>
  <div
    id="main-wrapper"
    class="wave-top"
  >
    <Toolbar />
    <b-container id="main-container">
      <b-row>
        <b-col
          align-self="center"
          class="splash-text"
        >
          <h1 class="text-center">
            Better habits 
            <!-- <img src="@/assets/logo.png" width="180" height="180" /> -->
          </h1>
        </b-col>
      </b-row>
      <b-row
        v-if="notify.checkPermission() == 2"
        align-h="center"
      >
        <b-col
          cols="4"
          style="text-align: center;"
        >
          <b-button
            variant="success"
            @click="notify.requestPermission()"
          >
            Enable notifications
          </b-button>
        </b-col>
      </b-row>
      <b-row
        v-if="showBreakButton"
        align-h="center"
      >
        <b-col
          cols="4"
          style="text-align: center;"
        >
          <b-button
            variant="success"
            @click="startBreak"
          >
            Start break
          </b-button>
        </b-col>
      </b-row>
      <b-row align-h="center">
        <b-col
          cols="4"
        >
          <TimePicker @input="updateLimits" />
        </b-col>
      </b-row>
      <b-row align-h="center">
        <b-col
          cols="4"
        >
          <hr>
          <div class="timepicker-container">
            Time until next break
            <div class="timepicker-input-wrapper">
              <input
                class="timepicker-input"
                type="text"
                :value="Math.floor(shortTimerCountdown / 60000)"
                disabled
                size="2"
              > :
              <input
                class="timepicker-input"
                type="text"
                :value="Math.floor(shortTimerCountdown / 1000) % 60"
                disabled
                size="2"
              >
            </div>
          </div>
        </b-col>
      </b-row>
    </b-container>
    <svg
      xmlns="http://www.w3.org/2000/svg"
      viewBox="0 0 1440 320"
    ><path
      fill="#ff9a0d"
      fill-opacity="1"
      d="M0,224L60,234.7C120,245,240,267,360,256C480,245,600,203,720,208C840,213,960,267,1080,288C1200,309,1320,299,1380,293.3L1440,288L1440,320L1380,320C1320,320,1200,320,1080,320C960,320,840,320,720,320C600,320,480,320,360,320C240,320,120,320,60,320L0,320Z"
    /></svg>
  </div>
</template>

<script lang="ts">
import LoginButton from '@/components/LoginButton.vue';
import Toolbar from '@/layouts/Toolbar.vue';
import { defineComponent, onMounted, Ref, ref, watch } from '@vue/composition-api';
import Notify, { PermissionAccess } from '@/utils/Notify';
import TimePicker from '@/components/TimePicker.vue';
import { chickResponses } from '@/models/Resource';
import { countdownInterval } from '@/utils/utils';
import IntervalController, { IntervalSignal } from '@/utils/IntervalController';
import { TimeHandler, WorkHours } from '@/utils/TimeHandler';
const notificationSound = require('../assets/notification-sound.mp3');

const defaultTimes = {
  workHours: {
    startHour: 8,
    startMin: 0,
    endHour: 17,
    endMin: 0
  },
  shortIntervalDuration: 20 * 60 * 1000,
  breakReminderDuration: 2 * 60 * 1000,
};

export default defineComponent({
  name: 'MainMenu',
  components: {
    LoginButton,
    Toolbar,
    TimePicker
  },
  setup() {
    let icon = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAYAAABzenr0AAAEKElEQVRYhbWUXUxTZxzGD4xMMgk3cmG2cMHkwmyLIcuMbJrYmCxLoS/sw+nM4i52YWTROHcxWRYXYnazG0dINq3uQp0wQ2fAFHFmQBEubOkpWBaMWuF8FE7f0563gEU+0o9nF3BKC4W2UN7kSZqTc/6/3/uct4fjNrDAk91wkUqAywO4PDhM74MnuzcyK3u4zVAAnrSAJxHwpBs86QFPInCa/oLNULB1YHD5GDAdXQKGwROsSBg86YGLHEEDl59buLv6HfAmPgU0dZzEiQHydm7gg8QIFwllDF9OCK7qjzYHH6j+ADyZ2wBczyz4qn0bg9uNxeCJtAm4HgE2Q1H2Aq7qn3MA1/NTdnDBUAieBPUBUYcJs31VGQNn+6oQdZgSr/kx8vmrmQvwNdX6wxGHCRcMpagrL4HbfCAJFO07hGjfoaRrbvMB1JWX4IKhFJFECWfNh1nUT+L1v3xgxMnyEpwo24GO+gqAJ4g5qjB3rRQzjUWYaSzC3LVSxByLDVnrK3CibAdOlpfg5QNjglzN+SwaMN1M2tXl/bCeq8Bc/yJkoW1PHK5noW0PwBPM9VfBeq4C7sv7V76aPzIXcJpurfeO55t3rRKYb96V7my0ZNOAZb1h4fuVmGkqXhZoKkb4fmUaAZMlPRjIM7ferY86a/9Ld9JjdiOitoOI2g4iZjem2z2iztphc2vnqTXhEmNvXbF0fHO11Yqw8+Nc/f+XW3N+gqsWa8x86593U+5coMx9xdIZsrTfyDlcj6X9BsytHXdkyo4mCXi92hsSZWjveQhb58UtE+jp/BXm1ruxZ7JPFvwzO+MCo0rgPYkyDD0R4O76ccsEhrrO409rNyTKICpTy69C9ml7JcogUQY//8OWCYz2nkGX/REkyiD7tL3LB1BV39QF2PPb8Qfmh88iOngsZwLj/XUYGRuHRBkonSqLC9hstgKJsplFMwWRoeMIepoh0QAmn13fFHR++FtMP72E8KOvMTn4PZY2GgLwStJBlFV2T29BViaQ+DuxhZjrMNhzCxbcpxJANZh++jtifG0SPOb6DBPjI0uzNHgVcXGmyjpXfwfUya906MoktqCNtscH+oV/seA+DTZ6GxJlePGkKUkg6GlJOW9M0Y6vEvAA22TK5FQPyMoEIoPHEHr8S8qBeryKF5HBL5eq/w4SDaS4Lyh6PJ5tKb+GciD4xVrDA2P3IPuUdQUkyqCN3llR/Yr42JGUcH2JVLuZDrJ+NKhi7xpw7fq6cI7jOAEoFH1a7+YkUqZbEITCtAIcx3GU0u0i1dpyBRd97G9FUV7LCK6vhoaGfEFhZyXKQhsGU/ZCotppAHlZwROXHAi8LlHWKFI2nQV4SqSBi4LfvzM9IcNFKd0u+dlhyaddkimzi1RTZTUYllUWlqhGZcoeymrwN686+Wk2df8PyVq091GZYo0AAAAASUVORK5CYII=';
    let notify = new Notify(icon);
    let longTimerCountdown = ref(0);
    let shortTimerCountdown = ref(0);
    let breakTimerCountdown = ref(0);
    let showBreakButton = ref(false);

    let playSound = () => {
      let audio = new Audio(notificationSound);
      audio.play();
    };

    let onTimeout = () => {
      console.log('hello')
      notify.notify('Time for a short break', 'Take a few seconds to look away from the screen and shake your legs', 0.5 * 60 * 1000);
      playSound();
    }

    let timer = new TimeHandler(
      defaultTimes.shortIntervalDuration,
      onTimeout.bind(this)
    )

    timer.attachTimeCounter((remainingTime) => shortTimerCountdown.value = remainingTime);

    onMounted(() => timer.start());

    let updateLimits = (limits: string[]) => {
      let [startHour, startMin, endHour, endMin, shortBreak] = limits.map((val) => parseInt(val));
      timer.setWorkHours({ startHour, startMin, endHour, endMin });
      timer.setDuration(shortBreak * 60 * 1000);
    };

    return {
      notify,
      updateLimits,
      shortTimerCountdown,
      longTimerCountdown,
      breakTimerCountdown,
      playSound,
      showBreakButton,
    };
  }
});
</script>

<style lang="scss">
.board-button {
  margin-left: 5px;
}

.countdown {
  border: 2px solid rgb(24, 24, 24);
  border-radius: 5px;
  margin: 20px auto;
  padding: 10px 0px;
}
</style>