<template>
    <div id="wrapper">
        <div id="tee-wrapper" class="tee-wrapper"></div>

        <template v-if="!loaded">
            <div>loading...</div>
        </template>

        <Transition name="slide-fade">
            <div v-if="loaded">
                <div class="variants ma-n2">
                    <template v-for="(p, pIdx) in currentPersonalities">
                        <button v-bind="getButtonProps(pIdx)">
                            {{ p.clientName }}
                        </button>
                    </template>
                </div>

                <div class="ma-2">{{ totalCorrectAnswers }} / {{ totalAnswers }}</div>
            </div>
        </Transition>



        <Transition name="slide-fade">
            <div v-if="state === GameState.Answered">
                <button class="button_next">
                    Next
                </button>

                <div class="text-caption">...or press SPACE</div>
            </div>
        </Transition>
    </div>
</template>

<script setup lang="ts">
import { renderer } from 'tee-skin-renderer';
import { ref, type ButtonHTMLAttributes } from 'vue';
import axios from 'axios';
import 'tee-skin-renderer/css';

enum GameState {
    Loading,
    Loaded,
    Answered,
}

interface ClientPersonality {
    clientSkin: string;
    clientName: string;
    clientSkinColorBody: number;
    clientSkinColorFeet: number;
    playtimeSeconds: number;
}

const state = ref<GameState>(GameState.Loading);
const teeContainer = ref<renderer.TeeContainer | null>(null);
const currentPersonalities = ref<ClientPersonality[]>([]);
const currentPersonalityIdx = ref<number | null>(null);
const answeredPersonalityIdx = ref<number | null>(null);
const loaded = ref<boolean>(false);
const totalAnswers = ref<number>(0);
const totalCorrectAnswers = ref<number>(0);

const randomInteger = (min: number, max: number): number => Math.floor(Math.random() * (max - min + 1)) + min;
const getSkinUrlBySkinName = (skin: string) => `https://skins.scrumplex.net/skin/${skin}.png`;
const getRandomPersonalitiesAsync = async (limit: number = 4) => {
    return (
        await axios.get<ClientPersonality[]>('https://tee.skin/api/skins/personalities/random', {
            params: {
                limit: limit,
                minimumPlaytimeSeconds: 180000,
            },
        })
    ).data;
};

const setTeeRandomPersonalityAsync = (personalities: ClientPersonality[]) => {
    return new Promise<number>((resolve, reject) => {
        try {
            const personalityIndex = randomInteger(0, personalities.length - 1);
            const personality = personalities[personalityIndex];

            if (personality.clientSkin === 'default'
                && personality.clientSkinColorBody === 0
                && personality.clientSkinColorFeet === 0
            ) {
                reject();
                return;
            }

            const tee = teeContainer.value!.tee;

            tee.addEventListener('tee:skin-loaded', (e) => {
                if (e.detail.payload.success) {
                    tee.colorBody = personality.clientSkinColorBody;
                    tee.colorFeet = personality.clientSkinColorFeet;
                    tee.useCustomColor = personality.clientSkinColorBody !== 0 || personality.clientSkinColorFeet !== 0;

                    resolve(personalityIndex);
                } else {
                    reject();
                }
            }, {
                once: true,
            });

            tee.skinUrl = getSkinUrlBySkinName(personality.clientSkin);
        } catch (error) {
            reject(error);
        }
    });
};

const setNewPersonalities = (callback: Function | undefined = undefined) => {
    getRandomPersonalitiesAsync().then((personalities) => {
        setTeeRandomPersonalityAsync(personalities).then((personalityIndex) => {
            currentPersonalities.value = personalities;
            currentPersonalityIdx.value = personalityIndex;
            state.value = GameState.Loaded;

            if (callback) {
                callback();
            }
        }).catch(() => {
            setNewPersonalities(callback);
        });
    });
}

const handleClickAnswer = (personalityIndex: number) => {
    if (state.value === GameState.Answered) {
        return;
    }

    state.value = GameState.Answered;
    answeredPersonalityIdx.value = personalityIndex;

    totalAnswers.value++;
    totalCorrectAnswers.value += +(currentPersonalityIdx.value === personalityIndex);
};

const getButtonProps = (personalityIndex: number): ButtonHTMLAttributes => {
    return {
        onClick: () => handleClickAnswer(personalityIndex),
        class: {
            'ma-2': true,
            'button_correct': state.value === GameState.Answered
                && personalityIndex === currentPersonalityIdx.value,
            'button_wrong': state.value === GameState.Answered
                && personalityIndex === answeredPersonalityIdx.value
                && personalityIndex !== currentPersonalityIdx.value,
        },
    };
}

renderer.createAsync({ skinUrl: 'https://skins.scrumplex.net/skin/default.png' }).then((container) => {
    teeContainer.value = container;
    document.getElementById('tee-wrapper')!.appendChild(container);

    setNewPersonalities(() => {
        loaded.value = true;
    });
});

document.addEventListener('keydown', (e) => {
    if (e.code === "Space") {

    }
});

</script>

<style scoped></style>
