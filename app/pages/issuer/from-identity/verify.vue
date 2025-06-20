<template>
  <div class="flex items-center bg-gray-100">
    <nav class="text-sm sm:text-base bg-white p-2 lg:p-4 rounded-md shadow-lg">
      <ol class="flex list-none p-0">
        <li class="flex items-center">
          <NuxtLink to="/verifier"
            class="mr-2 flex items-center text-gray-600 hover:text-blue-500 transition-colors duration-300">
            <svg xmlns="http://www.w3.org/2000/svg" height="1em" viewBox="0 0 576 512" class="cursor-pointer mr-1"
              fill="currentColor">
              <path
                d="M575.8 255.5c0 18-15 32.1-32 32.1h-32l.7 160.2c0 2.7-.2 5.4-.5 8.1V472c0 22.1-17.9 40-40 40H456c-1.1 0-2.2 0-3.3-.1c-1.4 .1-2.8 .1-4.2 .1H416 392c-22.1 0-40-17.9-40-40V448 384c0-17.7-14.3-32-32-32H256c-17.7 0-32 14.3-32 32v64 24c0 22.1-17.9 40-40 40H160 128.1c-1.5 0-3-.1-4.5-.2c-1.2 .1-2.4 .2-3.6 .2H104c-22.1 0-40-17.9-40-40V360c0-.9 0-1.9 .1-2.8V287.6H32c-18 0-32-14-32-32.1c0-9 3-17 10-24L266.4 8c7-7 15-8 22-8s15 2 21 7L564.8 231.5c8 7 12 15 11 24z" />
            </svg>
            Home
          </NuxtLink>
          <span class="mx-2">/</span>
        </li>
        <li class="flex items-center">C4DT Hands-On Workshop on E-ID</li>
      </ol>
    </nav>
  </div>
  <div class="flex mt-14 shadow-md rounded-xl bg-gray-50 min-h-[500px]">
    <div class="flex-1 mt-10 ml-10">
      <div v-if="step === Step.VERIFICATION_FORM" class="flex items-center justify-center gap-6">
        <form @submit.prevent="createVerificationRequest">
          <header class="pb-6 border-b border-gray-200">
            <h2 class="text-base font-semibold text-gray-900">
              Obtain Your Workshop Credential
            </h2>
            <p class="mt-1 text-sm text-gray-600">
              First step: Verify your identity
            </p>
          </header>
          <div class="grid grid-cols-1 sm:grid-cols-6 gap-x-6 gap-y-8">
            <div class="sm:col-span-4">
              <p class="text-red-500 mt-2 text-sm">
                We ask for the following data:
              </p>
              <div class="flex flex-col gap-2">
                <label class="inline-flex items-center gap-2 cursor-pointer">
                  - Given Name & Surname
                </label>
                <label class="inline-flex items-center gap-2 cursor-pointer">
                  - Birth date
                </label>
                <label class="inline-flex items-center gap-2 cursor-pointer">
                  - BETA-ID number
                </label>
              </div>
            </div>
          </div>
          <div class="mt-6 flex items-center justify-end gap-x-6">
            <button type="submit" :disabled="submitting"
              class="rounded-md bg-indigo-600 px-3 py-2 text-sm font-semibold text-white shadow hover:bg-indigo-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-600">
              {{
                submitting ? "Generating..." : "Start the verification process!"
              }}
            </button>
          </div>
        </form>
      </div>
      <div v-if="step === Step.VERIFICATION_CREATED" class="flex flex-col items-center text-center">
        <span>Please scan this QR code to accept the verification request!</span>
        <qrcode-vue class="mx-auto my-4" :value="verificationURL" level="H" size="250" render-as="svg" />
        <div class="break-all">
          <div>
            Verification ID:
            <span class="font-mono">{{ verificationId }}</span>
          </div>
          <span>Verification URL:</span>
          <span class="font-mono text-xs bg-gray-50 rounded-md p-2 max-w-xl mx-auto break-all select-all">
            {{ verificationURL }}
          </span>
        </div>
      </div>
      <div v-if="step === Step.VERIFICATION_SUCCEEDED">
        <div class="text-green-600 text-center">
          <h2 class="text-xl font-semibold mb-2">Verification Succeeded!</h2>
          <p class="text-sm">
            The holder has successfully verified their credential.
          </p>
        </div>
        <KeyValueDisplay :data="disclosedData" />
        <div class="mt-6 flex items-center justify-center gap-x-6">
          <button @click="redirectToCredentialGeneration" type="button"
            class="rounded-md bg-indigo-600 px-3 py-2 text-sm font-semibold text-white shadow hover:bg-indigo-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-600">
            Generate Credential for this identity?
          </button>
        </div>
      </div>
      <div v-if="step === Step.VERIFICATION_FAILED" class="text-red-600 text-center">
        <h2 class="text-xl font-semibold mb-2">Verification Failed!</h2>
        <p class="text-sm">The holder failed to verify their credential.</p>
      </div>
    </div>
    <aside class="w-72 bg-gray-300 p-6 -mt-10 rounded-md shadow-lg">
      <h2 class="text-xl font-semibold mb-2">Updates</h2>
      <ol class="list-decimal ml-4 space-y-1">
        <li v-for="msg in logMessages" :key="msg.message + msg.source">
          [{{ msg.source }}<span v-if="msg.target">&rarr; {{ msg.target }}</span>] {{ msg.message }}
        </li>
      </ol>
    </aside>
  </div>
</template>
<script setup lang="ts">
import QrcodeVue from "qrcode.vue";

import type { ActionLog } from "~/services/VerifiableCredential";
import {
  createSwiyuBetaIDVerification,
  checkVerificationStatus,
} from "~/services/swiyu/index";
const logMessages = ref<ActionLog[]>([]);

function addToLog(
  message: string,
  source: string = "Issuer",
  target?: string
): void {
  logMessages.value.push({ source, target, message });
}

enum Step {
  VERIFICATION_FORM,
  VERIFICATION_CREATED,
  VERIFICATION_SUCCEEDED,
  VERIFICATION_FAILED,
}
const step = ref<Step>(Step.VERIFICATION_FORM);
const verificationURL = ref<string>("");
const verificationId = ref<string>("");
const submitting = ref(false);
const verificationRequestStatus = ref<string>("PENDING");

async function createVerificationRequest(): Promise<void> {
  submitting.value = true;
  ({
    verificationId: verificationId.value,
    verificationURL: verificationURL.value,
  } = await createSwiyuBetaIDVerification());
  step.value = Step.VERIFICATION_CREATED;
  addToLog("Verification request created successfully.", "Verifier");
  // Here you would typically display the QR code or link to the user
  // For example, you could use a QR code component to show the verification URL
  console.log("Verification URL:", verificationURL.value);
  submitting.value = false;
}

const disclosedData = ref<Record<string, any>>({});
onMounted(() => {
  const checkStatusInterval = setInterval(async () => {
    if (verificationId.value) {
      let status = null;
      ({ status, disclosedData: disclosedData.value } =
        await checkVerificationStatus(verificationId.value));
      if (status !== verificationRequestStatus.value) {
        verificationRequestStatus.value = status;
      } else {
        console.log("No status change detected.");
        return;
      }
      if (status === "SUCCESS") {
        clearInterval(checkStatusInterval);
        addToLog("Verification request was fulfiled!");
        step.value = Step.VERIFICATION_SUCCEEDED;
      } else if (status === "FAILED") {
        clearInterval(checkStatusInterval);
        addToLog("Verification request failed to be fulfilled!");
        step.value = Step.VERIFICATION_FAILED;
      } else {
        console.log("Still figuring it out...");
      }
    }
  }, 2500);
});

function redirectToCredentialGeneration(): void {
  console.log(disclosedData.value);
  navigateTo({
    path: "/issuer/from-identity/issue",
    query: {
      firstName: disclosedData.value.credential_subject_data.given_name,
      lastName: disclosedData.value.credential_subject_data.family_name,
      birthDate: disclosedData.value.credential_subject_data.birth_date,
      betaIdDocumentNumber: disclosedData.value.credential_subject_data.document_number,
      betaIdCnf: JSON.stringify(disclosedData.value.credential_subject_data.cnf),
    }
  });
  console.log("Redirecting to credential generation...");
}

definePageMeta({ layout: "leo-inc" });
useHead({
  title: "Leo Inc. HR - Employees",
  meta: [
    {
      name: "description",
      content:
        "Manage Employee Information and maintain their contact information.",
    },
  ],
});
</script>
