<script setup>
import { ref, computed, watch } from 'vue';
import { Inertia } from '@inertiajs/inertia';
import { useForm, usePage } from '@inertiajs/inertia-vue3';
import JetActionSection from '@/Jetstream/ActionSection.vue';
import JetButton from '@/Jetstream/Button.vue';
import JetDangerButton from '@/Jetstream/DangerButton.vue';
import JetInput from '@/Jetstream/Input.vue';
import JetInputError from '@/Jetstream/InputError.vue';
import JetLabel from '@/Jetstream/Label.vue';
import JetSecondaryButton from '@/Jetstream/SecondaryButton.vue';
import PaConfirmsPassword from '@/JetstreamPlus/Components/Auth/ConfirmsPassword.vue';

const props = defineProps({
    requiresConfirmation: Boolean,
});

const enabling = ref(false);
const confirming = ref(false);
const disabling = ref(false);
const qrCode = ref(null);
const setupKey = ref(null);
const recoveryCodes = ref([]);

const confirmationForm = useForm({
    code: '',
});

const twoFactorEnabled = computed(
    () => ! enabling.value && usePage().props.value.user?.two_factor_enabled,
);

watch(twoFactorEnabled, () => {
    if (! twoFactorEnabled.value) {
        confirmationForm.reset();
        confirmationForm.clearErrors();
    }
});

const enableTwoFactorAuthentication = () => {
    enabling.value = true;

    Inertia.post('/user/two-factor-authentication', {}, {
        preserveScroll: true,
        onSuccess: () => Promise.all([
            showQrCode(),
            showSetupKey(),
            showRecoveryCodes(),
        ]),
        onFinish: () => {
            enabling.value = false;
            confirming.value = props.requiresConfirmation;
        },
    });
};

const showQrCode = () => {
    return axios.get('/user/two-factor-qr-code').then(response => {
        qrCode.value = response.data.svg;
    });
};

const showSetupKey = () => {
    return axios.get('/user/two-factor-secret-key').then(response => {
        setupKey.value = response.data.secretKey;
    });
}

const showRecoveryCodes = () => {
    return axios.get('/user/two-factor-recovery-codes').then(response => {
        recoveryCodes.value = response.data;
    });
};

const confirmTwoFactorAuthentication = () => {
    confirmationForm.post('/user/confirmed-two-factor-authentication', {
        errorBag: "confirmTwoFactorAuthentication",
        preserveScroll: true,
        preserveState: true,
        onSuccess: () => {
            confirming.value = false;
            qrCode.value = null;
            setupKey.value = null;
        },
    });
};

const regenerateRecoveryCodes = () => {
    axios
        .post('/user/two-factor-recovery-codes')
        .then(() => showRecoveryCodes());
};

const disableTwoFactorAuthentication = () => {
    disabling.value = true;

    Inertia.delete('/user/two-factor-authentication', {
        preserveScroll: true,
        onSuccess: () => {
            disabling.value = false;
            confirming.value = false;
        },
    });
};
</script>

<template>
    <JetActionSection>
        <template #title>
            {{$t('system.auth.twoFactorAuth.title')}}
        </template>

        <template #description>
            {{$t('system.auth.twoFactorAuth.description')}}
        </template>

        <template #content>
            <h3 v-if="twoFactorEnabled && ! confirming" class="text-lg font-medium text-gray-900">
                {{$t('system.auth.twoFactorAuth.enabled.title')}}
            </h3>

            <h3 v-else-if="twoFactorEnabled && confirming" class="text-lg font-medium text-gray-900">
                {{$t('system.auth.twoFactorAuth.finish.title')}}
            </h3>

            <h3 v-else class="text-lg font-medium text-gray-900">
                {{$t('system.auth.twoFactorAuth.disabled.title')}}
            </h3>

            <div v-if="!twoFactorEnabled" class="mt-3 max-w-xl text-sm text-gray-600">
                <p>
                    {{$t('system.auth.twoFactorAuth.disabled.description')}}
                </p>
            </div>

            <div v-if="twoFactorEnabled">
                <div v-if="qrCode">
                    <div class="mt-4 max-w-xl text-sm text-gray-600">
                        <p v-if="confirming" class="font-semibold">
                            {{$t('system.auth.twoFactorAuth.finish.description')}}
                        </p>

                        <p v-else>
                            {{$t('system.auth.twoFactorAuth.enabled.description')}}
                        </p>
                    </div>

                    <div class="mt-4" v-html="qrCode" />

                    <div class="mt-4 max-w-xl text-sm text-gray-600" v-if="setupKey">
                        <p class="font-semibold">
                            {{$t('system.auth.twoFactorAuth.code.setupKey')}}: <span v-html="setupKey"></span>
                        </p>
                    </div>

                    <div v-if="confirming" class="mt-4">
                        <JetLabel for="code" :value="$t('system.auth.twoFactorAuth.code.label')" />

                        <JetInput
                            id="code"
                            v-model="confirmationForm.code"
                            type="text"
                            name="code"
                            class="block mt-1 w-1/2"
                            inputmode="numeric"
                            autofocus
                            autocomplete="one-time-code"
                            @keyup.enter="confirmTwoFactorAuthentication"
                        />

                        <JetInputError :message="confirmationForm.errors.code" class="mt-2" />
                    </div>
                </div>

                <div v-if="recoveryCodes.length > 0 && ! confirming">
                    <div class="mt-4 max-w-xl text-sm text-gray-600">
                        <p class="font-semibold">
                            {{$t('system.auth.twoFactorAuth.enabled.info')}}
                        </p>
                    </div>

                    <div class="grid gap-1 max-w-xl mt-4 px-4 py-4 font-mono text-sm bg-gray-100 rounded-lg">
                        <div v-for="code in recoveryCodes" :key="code">
                            {{ code }}
                        </div>
                    </div>
                </div>
            </div>

            <div class="mt-5">
                <div v-if="! twoFactorEnabled">
                    <PaConfirmsPassword
                        @confirmed="enableTwoFactorAuthentication">
                        <JetButton type="button" :class="{ 'opacity-25': enabling }" :disabled="enabling">
                            {{$t('system.global.action.enable')}}
                        </JetButton>
                    </PaConfirmsPassword>
                </div>

                <div v-else>
                    <PaConfirmsPassword
                        @confirmed="confirmTwoFactorAuthentication"
                    >
                        <JetButton
                            v-if="confirming"
                            type="button"
                            class="mr-3"
                            :class="{ 'opacity-25': enabling }"
                            :disabled="enabling"
                        >
                            {{$t('system.global.action.confirm')}}
                        </JetButton>
                    </PaConfirmsPassword>

                    <PaConfirmsPassword @confirmed="regenerateRecoveryCodes">
                        <JetSecondaryButton
                            v-if="recoveryCodes.length > 0 && ! confirming"
                            class="mr-3"
                        >
                            {{$t('system.auth.twoFactorAuth.code.regenerateRecoveryCodes')}}
                        </JetSecondaryButton>
                    </PaConfirmsPassword>

                    <PaConfirmsPassword @confirmed="showRecoveryCodes">
                        <JetSecondaryButton
                            v-if="recoveryCodes.length === 0 && ! confirming"
                            class="mr-3"
                        >
                            {{$t('system.auth.twoFactorAuth.code.showRecoveryCodes')}}
                        </JetSecondaryButton>
                    </PaConfirmsPassword>

                    <PaConfirmsPassword @confirmed="disableTwoFactorAuthentication">
                        <JetSecondaryButton
                            v-if="confirming"
                            :class="{ 'opacity-25': disabling }"
                            :disabled="disabling"
                        >
                            {{$t('system.global.action.cancel')}}
                        </JetSecondaryButton>
                    </PaConfirmsPassword>

                    <PaConfirmsPassword @confirmed="disableTwoFactorAuthentication">
                        <JetDangerButton
                            v-if="! confirming"
                            :class="{ 'opacity-25': disabling }"
                            :disabled="disabling"
                        >
                            {{$t('system.global.action.disable')}}
                        </JetDangerButton>
                    </PaConfirmsPassword>
                </div>
            </div>
        </template>
    </JetActionSection>
</template>
