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
                            <div class="tee" data-skin="https://skins.scrumplex.net/skin/default.png">
                            </div>
                            {{ p.clientName }}
                        </button>
                    </template>
                </div>

                <div class="mt-4">
                    <div class="ma-n2">
                        <div class="ma-2 stats-badge stats-badge_correct">
                            {{ totalCorrectAnswers }}
                        </div>

                        <div class="ma-2 stats-badge stats-badge_wrong">
                            {{ totalAnswers - totalCorrectAnswers }}
                        </div>
                    </div>
                </div>
            </div>
        </Transition>

        <div v-bind:class="{
            'container-next': true,
            'container-next_visible': state === GameState.Answered
        }">
            <button class="button_next" v-on:click="handleNext">
                Next
            </button>

            <div class="text-caption">...or press SPACE</div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { renderer } from 'tee-skin-renderer';
import { nextTick, ref, type ButtonHTMLAttributes } from 'vue';
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
    state.value = GameState.Loading;

    document.querySelectorAll<renderer.TeeContainer>('.variants .tee.tee_initialized').forEach((container) => {
        container.tee.addEventListener('tee:skin-loaded', (e) => {
            if (e.detail.payload.success) {
                e.detail.tee.useCustomColor = false;
            }
        }, {
            once: true,
        });

        container.tee.skinUrl = getSkinUrlBySkinName('default');
    });

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
};

const handleClickAnswer = (personalityIndex: number) => {
    if (state.value !== GameState.Loaded) {
        return;
    }

    state.value = GameState.Answered;
    answeredPersonalityIdx.value = personalityIndex;

    totalAnswers.value++;
    totalCorrectAnswers.value += +(currentPersonalityIdx.value === personalityIndex);

    document.querySelectorAll<renderer.TeeContainer>('.variants .tee').forEach((container, index) => {
        const tee = container.tee;
        const personality = currentPersonalities.value[index];

        tee.skinUrl = getSkinUrlBySkinName(personality.clientSkin);
        tee.colorBody = personality.clientSkinColorBody;
        tee.colorFeet = personality.clientSkinColorFeet;
        tee.useCustomColor = personality.clientSkinColorBody !== 0 || personality.clientSkinColorFeet !== 0;
    });
};

const handleNext = () => {
    if (state.value !== GameState.Answered) {
        return;
    }

    setNewPersonalities(() => {
        answeredPersonalityIdx.value = null;
    });
};

const getButtonProps = (personalityIndex: number): ButtonHTMLAttributes => {
    return {
        onClick: () => handleClickAnswer(personalityIndex),
        disabled: state.value === GameState.Loading,
        class: {
            'ma-2': true,
            'button_correct': state.value === GameState.Answered
                && personalityIndex === currentPersonalityIdx.value,
            'button_wrong': state.value === GameState.Answered
                && personalityIndex === answeredPersonalityIdx.value
                && personalityIndex !== currentPersonalityIdx.value,
        },
    };
};

renderer.createAsync({ skinUrl: 'https://skins.scrumplex.net/skin/default.png' }).then((container) => {
    teeContainer.value = container;
    document.getElementById('tee-wrapper')!.appendChild(container);

    setNewPersonalities(() => {
        loaded.value = true;
        nextTick(() => {
            renderer.initializeAsync();
        });
    });
});

document.addEventListener('keydown', (e) => {
    if (e.key === ' ') {
        handleNext();
    }

    switch (e.key) {
        case '1':
            handleClickAnswer(0);
            break;

        case '2':
            handleClickAnswer(1);
            break;

        case '3':
            handleClickAnswer(2);
            break;

        case '4':
            handleClickAnswer(3);
            break;
    }
});

</script>
