<script setup lang="ts">
import type { ILcdControllerConfigs } from "@/services/types/devices";
import { computed, ref } from "vue";
import PlusIcon from "../icons/PlusIcon.vue";
import MinusIcon from "../icons/MinusIcon.vue";
import ControllerLabel from "./ControllerLabel.vue";
import { vOnLongPress } from "@vueuse/components";

const emit = defineEmits<{
    (event: "changevalue", value: number): void;
}>();
const props = defineProps<{
    controller: ILcdControllerConfigs;
    invert: boolean;
}>();
/** One digit per figure of the maximum value (at most 3), so a 0-64 preset doesn't show a spare "8". */
const digits = computed(() => Math.min(3, Math.max(1, String(props.controller.maxValue || 127).length)));
const digitsWidth = computed(() => `${digits.value * 16}px`);
const currentValue = ref<string>(formatOutput(props.controller.value || props.controller.minValue || 1));
const delay = 1000;
const longWait = 1500;
const wait = 500;
const shortWait = 250;

const longPressedHook = ref(false);
let step = 1;
let interval: NodeJS.Timeout | null = null;

function resetHook() {
    if (interval && longPressedHook.value) {
        longPressedHook.value = false;
        step = 1;
        clearInterval(interval);
        interval = null;
        changeValue(parseInt(currentValue.value));
    }
}

function onLongPressCallbackHook(e: Event, callback: CallableFunction) {
    longPressedHook.value = true;
    setTimeout(() => {
        step = 10;
        setTimeout(() => {
            if (interval) {
                clearInterval(interval);
                interval = setInterval(() => callback(e, false), shortWait);
            }
        }, longWait);
    }, longWait);
    callback(e, false);
    interval = setInterval(() => callback(e, false), wait);
}

function formatOutput(value: number) {
    return value + "";
}

function incrementValue(e: Event, emit = true) {
    const maxValue = props.controller.maxValue || 127;
    const oldValue = parseInt(currentValue.value);
    let newValue = oldValue + step;
    newValue = newValue < maxValue ? newValue : maxValue;
    if (newValue <= maxValue && newValue !== oldValue) {
        changeValue(newValue, emit);
    } else {
        resetHook();
    }
}

function decrementValue(e: Event, emit = true) {
    const minValue = props.controller.minValue || 0;
    const oldValue = parseInt(currentValue.value);
    if (oldValue <= minValue) return;
    let newValue = oldValue - step;
    newValue = newValue > minValue ? newValue : minValue;
    if (newValue >= minValue && newValue !== oldValue) {
        changeValue(newValue, emit);
    } else {
        resetHook();
    }
}

let changeValueTimeout: number | null = null;

function changeValue(value: number, emitChange = true) {
    if (changeValueTimeout) clearTimeout(changeValueTimeout);
    setTimeout(() => {
        currentValue.value = formatOutput(value);
        if (emitChange) emit("changevalue", value);
    }, wait);
}
</script>

<template>
    <div class="flex flex-col items-center">
        <div
            class="flex items-center flex-row justify-center border-3d px-1 relative"
            :class="invert ? 'invert bg-white' : 'bg-black'"
            :style="{ width: `calc(${digitsWidth} + 32px)` }"
        >
            <div class="relative m-1" :style="{ width: digitsWidth }">
                <div
                    class="absolute inset-0 font-digital text-2xl select-none cursor-default pointer-events-none bg-black/0 text-off/40 text-right"
                    :class="{ invert: invert }"
                >
                    {{ "8".repeat(digits) }}
                </div>
                <input
                    type="text"
                    :min="controller.minValue || undefined"
                    :max="controller.maxValue || undefined"
                    v-model="currentValue"
                    readonly
                    :maxlength="digits"
                    class="relative w-full font-digital text-2xl select-none cursor-default pointer-events-none bg-black/0 text-on z-10 text-right"
                    :class="invert ? 'invert bg-black/0' : ''"
                />
            </div>
            <div class="flex flex-col items-center justify-between text-xs mt-0.5" :class="{ invert: !invert }">
                <button
                    type="button"
                    class="opacity-30 hover:opacity-100 transition-opacity"
                    @click="incrementValue"
                    v-on-long-press="[(e: Event) => onLongPressCallbackHook(e, incrementValue), { delay, modifiers: { prevent: true } }]"
                    @mouseout.prevent="resetHook"
                >
                    <PlusIcon class="select-none" />
                </button>
                <button
                    type="button"
                    class="opacity-30 hover:opacity-100 transition-opacity"
                    @click="decrementValue"
                    v-on-long-press="[(e: Event) => onLongPressCallbackHook(e, decrementValue), { delay, modifiers: { prevent: true } }]"
                    @mouseout.prevent="resetHook"
                >
                    <MinusIcon class="select-none" />
                </button>
            </div>
        </div>
        <ControllerLabel :invert="invert" :label="controller.label" />
    </div>
</template>

<style scoped lang="scss"></style>
