<template>
  <div id="app">
    <textarea v-model="content" rows="10" cols="100" />
    <br />
    <br />
    <button
      @click="send('SWITCH')"
    >{{ current.matches('raw') ? 'Show rendered Markdown' : 'Show HTML code' }}</button>
    <button
      @click="send('TOGGLE')"
    >{{ current.matches('hidden') ? 'Show preview' : 'Hide preview' }}</button>
    <transition name="fade" mode="out-in">
      <div v-html="rendered" v-if="current.matches('visible.rendered')" key="rendered" />
      <div v-if="current.matches('visible.raw')" key="raw">
        <pre>{{ raw }}</pre>
      </div>
    </transition>
  </div>
</template>

<script>
import MarkdownIt from 'markdown-it';
import { indent } from 'indent.js';
import { createMachine, State, interpret } from 'xstate';

const md = new MarkdownIt();

const toggleMachine = createMachine({
  id: 'toggle',
  initial: 'visible',
  states: {
    visible: {
      on: {
        TOGGLE: 'hidden'
      },
      initial: 'rendered',
      states: {
        rendered: {
          on: {
            SWITCH: 'raw'
          }
        },
        raw: {
          on: {
            SWITCH: 'rendered'
          }
        },
        memo: {
          type: 'history'
        }
      }
    },
    hidden: {
      on: {
        TOGGLE: 'visible.memo'
      }
    }
  }
});

const savedState = JSON.parse(localStorage.getItem('state'));
const previousState = State.create(savedState || toggleMachine.initialState);
const resolvedState = toggleMachine.resolveState(previousState);

export default {
  name: 'App',
  data() {
    return {
      content:
        '# Hello there!\n\n- Type some Markdown on the left\n- See HTML in the right\n- Magic\n\n![An orange jellyfish](https://i.picsum.photos/id/1069/400/250.jpg)',
      toggleService: interpret(toggleMachine),
      current: toggleMachine.initialState
    };
  },
  created() {
    this.toggleService
      .onTransition(state => {
        this.current = state;

        try {
          const state = JSON.stringify(this.current);
          localStorage.setItem('state', state);
        } catch {
          console.error('Local storage is unavailable.');
        }
      })
      .start(resolvedState);
  },
  computed: {
    rendered() {
      return md.render(this.content);
    },
    raw() {
      return indent.html(this.rendered, {
        tabString: '  '
      });
    }
  },
  methods: {
    send(event) {
      this.toggleService.send(event);
    }
  }
};
</script>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter, .fade-leave-to /* .fade-leave-active below version 2.1.8 */ {
  opacity: 0;
}
</style>