<template>
  <client-only>
    <div
      v-if="visible"
      class="fixed z-50 w-full xl:w-3/5 xl:right-2 bottom-0 xl:bottom-2 shadow-2xl p-3 bg-white rounded overflow-auto top-0 sm:top-auto"
    >
      <div v-if="!furtherSettingsOn">
        <!-- cookie info -->
        <div class="font-medium text-center">
          {{ $t(cookieGroups?.barTitle) }}
        </div>
        <div class="leading-relaxed pb-5">
          {{ $t(cookieGroups.barDescription) }}

          <SfLink :tag="NuxtLink" :to="localePath(paths.privacyPolicy)">
            {{ $t('CookieBar.Privacy Settings') }}
          </SfLink>
        </div>
        <!-- checkboxes -->
        <div v-if="cookieJson" class="grid sm:grid-cols-4 xs:grid-cols-3">
          <template v-for="(cookieGroup, index) in cookieJson.groups" :key="index">
            <div v-if="cookieGroup?.cookies?.length" class="sm:mb-5 mb-2 pr-2 flex items-center">
              <SfCheckbox
                :id="cookieGroup.name"
                v-model="cookieGroup.accepted"
                @update:model-value="triggerGroupConsent(cookieGroup)"
                :disabled="index === defaults.ESSENTIAL_COOKIES_INDEX"
              />
              <label class="ml-2 cursor-pointer peer-disabled:text-disabled-900" :for="cookieGroup.name">
                {{ $t(cookieGroup.name) }}
              </label>
            </div>
          </template>
        </div>
      </div>
      <div v-else class="overflow-y-auto h-80 pb-2">
        <template v-for="(cookieGroup, groupIndex) in cookieJson.groups" :key="groupIndex">
          <div v-if="cookieGroup?.cookies?.length" class="mb-2 bg-gray-100 p-2">
            <SfCheckbox
              class="align-text-top"
              :id="cookieGroup.name"
              v-model="cookieGroup.accepted"
              @update:model-value="triggerGroupConsent(cookieGroup)"
              :disabled="groupIndex === defaults.ESSENTIAL_COOKIES_INDEX"
            />
            <label
              class="ml-2 cursor-pointer peer-disabled:text-disabled-900 align-text-bottom font-medium"
              :for="cookieGroup.name"
            >
              {{ $t(cookieGroup.name) }}
            </label>
            <div class="leading-6 my-2">
              {{ $t(cookieGroup.description) }}
            </div>
            <div v-if="cookieGroup.showMore ?? false">
              <div v-for="(cookie, cookieIndex) in cookieGroup.cookies" :key="cookieIndex" class="mb-4">
                <div class="flex w-full items-center bg-white mb-1 p-2">
                  <SfCheckbox
                    class="ml-1"
                    :id="cookie.name"
                    v-model="cookie.accepted"
                    @update:model-value="triggerCookieConsent(cookieGroup)"
                    :disabled="groupIndex === defaults.ESSENTIAL_COOKIES_INDEX"
                  />
                  <label class="ml-2 cursor-pointer peer-disabled:text-disabled-900 font-medium" :for="cookie.name">
                    {{ cookie.name }}
                  </label>
                </div>
                <div v-for="propKey in Object.keys(cookie)" :key="propKey">
                  <div v-if="propKey !== 'name' && propKey !== 'accepted'" class="flex w-full mb-1 p-2 bg-white">
                    <div class="w-1/4">
                      {{ propKey }}
                    </div>
                    <div class="w-3/4">
                      <template v-if="propKey === 'PrivacyPolicy'">
                        <!-- TODO -->
                        <SfLink :tag="NuxtLink" :link="localePath(paths.privacyPolicy)">
                          {{ $t('CookieBar.Privacy Settings') }}
                        </SfLink>
                      </template>
                      <template v-else>
                        {{ cookie[propKey as keyof Cookie] }}
                      </template>
                    </div>
                  </div>
                </div>
              </div>
            </div>
            <SfLink v-if="!cookieGroup.showMore ?? false" href="#" size="sm" @click="cookieGroup.showMore = true">
              {{ $t('CookieBar.More information') }}
            </SfLink>
            <SfLink v-else href="#" size="sm" @click="cookieGroup.showMore = false">
              {{ $t('CookieBar.Show less') }}
            </SfLink>
          </div>
        </template>
      </div>
      <!-- further settings / back button -->
      <div class="text-center mt-2">
        <SfLink v-if="!furtherSettingsOn" href="#" @click="furtherSettingsOn = true">
          {{ $t('CookieBar.Further Settings') }}
        </SfLink>
        <SfLink v-else href="#" @click="furtherSettingsOn = false">
          {{ $t('CookieBar.Back') }}
        </SfLink>
      </div>
      <!-- action buttons -->
      <div class="w-full flex flex-col xl:flex-row mt-5 gap-2 mb-2">
        <div class="flex-1">
          <SfButton
            class="w-full"
            :aria-disabled="false"
            type="button"
            :aria-label="$t('CookieBar.Accept All')"
            @click="setAllCookiesState(true)"
          >
            {{ $t('CookieBar.Accept All') }}
          </SfButton>
        </div>
        <div class="flex-1">
          <SfButton
            class="w-full"
            :aria-disabled="false"
            type="button"
            :aria-label="$t('CookieBar.Reject All')"
            @click="setAllCookiesState(false)"
          >
            {{ $t('CookieBar.Reject All') }}
          </SfButton>
        </div>
        <div class="flex-1">
          <SfButton
            variant="secondary"
            class="w-full"
            :aria-disabled="false"
            type="button"
            :aria-label="$t('CookieBar.Accept Selection')"
            @click="setConsent()"
          >
            {{ $t('CookieBar.Accept Selection') }}
          </SfButton>
        </div>
      </div>
    </div>
    <!-- button to open cookie tab -->
    <SfButton
      v-else
      variant="secondary"
      class="z-10 fixed bottom-[4.3rem] md:bottom-2 left-2 xl:left-auto xl:right-2 bg-white !px-3"
      :aria-label="$t('CookieBar.Cookie Settings')"
      @click="changeVisibilityState"
    >
      <SfIconCheckBox />
    </SfButton>
  </client-only>
</template>

<script setup lang="ts">
import { SfLink, SfButton, SfCheckbox, SfIconCheckBox } from '@storefront-ui/vue';
import { Cookie, CookieGroup } from '~/cookie.config';

const NuxtLink = resolveComponent('NuxtLink');
const localePath = useLocalePath();
const runtimeConfig = useRuntimeConfig();
const cookieGroups = ref(runtimeConfig.public.cookieGroups);

const {
  initializeCookies,
  data: cookieJson,
  visible,
  setConsent,
  setAllCookiesState,
  changeVisibilityState,
} = useReadCookieBar();

initializeCookies();

const furtherSettingsOn = ref(false);

const triggerCookieConsent = (group: CookieGroup) => {
  group.accepted = group.cookies.some((cookie: Cookie) => cookie.accepted);
};

const triggerGroupConsent = (group: CookieGroup) => {
  group.cookies.forEach((cookie: Cookie) => {
    cookie.accepted = group.accepted;
  });
};
</script>
