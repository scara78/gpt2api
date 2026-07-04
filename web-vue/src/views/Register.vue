<template>
  <div class="register-page">
    <PagePanel class="space-y-4">
      <PanelHeader title="Înregistrare cont" align="start">
        <template #actions>
          <StateBadge :tone="registerConfig?.enabled ? 'success' : 'muted'" shape="rounded" size="sm">
            {{ registerConfig?.enabled ? 'Activ' : 'Oprit' }}
          </StateBadge>
          <Button
            size="sm"
            variant="primary"
            :disabled="legacySaving || !registerConfig || registerConfig.enabled"
            @click="saveLegacyConfig"
          >
            Salvează configurația
          </Button>
        </template>
      </PanelHeader>

      <PageLoadingState
        v-if="legacyLoading && !registerConfig"
        title="Se încarcă configurația de înregistrare"
        description="Se citesc sursele de email, parametrii sarcinii și starea de execuție."
      />

      <div v-else-if="registerConfig" class="register-layout">
        <div class="register-config-column">
          <FormSection title="Parametri sarcină" density="roomy">
            <div class="register-form-grid">
              <label class="register-field">
                <span class="register-label">Mod sarcină</span>
                <GroupedSelectMenu
                  v-model="registerConfig.mode"
                  :groups="registerModeGroups"
                  selected-indicator="none"
                  :disabled="registerConfig.enabled"
                  block
                />
              </label>

              <label v-if="registerConfig.mode === 'total'" class="register-field">
                <span class="register-label">Total înregistrări</span>
                <Input
                  v-model.number="registerConfig.total"
                  type="number"
                  min="1"
                  block
                  :disabled="registerConfig.enabled || registerConfig.mode !== 'total'"
                />
              </label>

              <label v-else-if="registerConfig.mode === 'quota'" class="register-field">
                <span class="register-label">Cotă rămasă țintă</span>
                <Input
                  v-model.number="registerConfig.target_quota"
                  type="number"
                  min="1"
                  block
                  :disabled="registerConfig.enabled"
                />
              </label>

              <label v-else class="register-field">
                <span class="register-label">Conturi disponibile țintă</span>
                <Input
                  v-model.number="registerConfig.target_available"
                  type="number"
                  min="1"
                  block
                  :disabled="registerConfig.enabled"
                />
              </label>

              <label class="register-field">
                <span class="register-label">Număr fire</span>
                <Input
                  v-model.number="registerConfig.threads"
                  type="number"
                  min="1"
                  block
                  :disabled="registerConfig.enabled"
                />
              </label>

              <label v-if="registerConfig.mode !== 'total'" class="register-field">
                <span class="register-label">Interval verificare (sec)</span>
                <Input
                  v-model.number="registerConfig.check_interval"
                  type="number"
                  min="1"
                  block
                  :disabled="registerConfig.enabled"
                />
              </label>

              <label class="register-field">
                <span class="register-label">Proxy înregistrare</span>
                <GroupedSelectMenu
                  :model-value="registerProxyMode"
                  :groups="registerProxyModeGroups"
                  selected-indicator="none"
                  :disabled="registerConfig.enabled"
                  block
                  @update:model-value="setRegisterProxyMode"
                />
              </label>

              <label v-if="registerProxyMode === 'group'" class="register-field">
                <span class="register-label">Grup proxy</span>
                <GroupedSelectMenu
                  :model-value="selectedRegisterProxyGroupId"
                  :groups="registerProxyGroupGroups"
                  selected-indicator="none"
                  :disabled="registerConfig.enabled"
                  block
                  @update:model-value="selectRegisterProxyGroup"
                />
              </label>

              <label v-else-if="registerProxyMode === 'custom'" class="register-field">
                <span class="register-label">Proxy personalizat</span>
                <Input
                  :model-value="customRegisterProxyInput"
                  block
                  root-class="font-mono"
                  placeholder="http://127.0.0.1:7890"
                  :disabled="registerConfig.enabled"
                  @update:model-value="setCustomRegisterProxyInput"
                />
              </label>

              <p class="register-proxy-hint register-field--full">
                {{ registerProxyHint }}
              </p>
            </div>
          </FormSection>

          <FormSection title="Cereri email" density="roomy">
            <div class="register-form-grid register-form-grid--mail">
              <label class="register-field">
                <span class="register-label">Timeout cerere (sec)</span>
                <Input
                  v-model.number="registerConfig.mail.request_timeout"
                  type="number"
                  min="1"
                  block
                  :disabled="registerConfig.enabled"
                />
              </label>

              <label class="register-field">
                <span class="register-label">Așteptare cod verificare (sec)</span>
                <Input
                  v-model.number="registerConfig.mail.wait_timeout"
                  type="number"
                  min="1"
                  block
                  :disabled="registerConfig.enabled"
                />
              </label>

              <label class="register-field">
                <span class="register-label">Interval polling (sec)</span>
                <Input
                  v-model.number="registerConfig.mail.wait_interval"
                  type="number"
                  min="1"
                  step="0.2"
                  block
                  :disabled="registerConfig.enabled"
                />
              </label>

              <label class="register-field register-field--full">
                <span class="register-label">User-Agent cerere</span>
                <Input
                  v-model.trim="registerConfig.mail.user_agent"
                  block
                  root-class="font-mono"
                  placeholder="UA browser implicit"
                  :disabled="registerConfig.enabled"
                />
              </label>
            </div>
          </FormSection>

          <FormSection title="Surse email" density="roomy">
            <template #actions>
              <MetaChip v-if="enabledProviderIssueCount" size="xs" tone="danger">
                Lipsă {{ enabledProviderIssueCount }}
              </MetaChip>
              <MetaChip size="xs" tone="muted">Activate {{ enabledProviderCount }} / {{ registerProviders.length }}</MetaChip>
              <Button
                size="sm"
                variant="outline"
                :disabled="registerConfig.enabled"
                @click="addProvider"
              >
                Adaugă sursă
              </Button>
            </template>

            <div class="register-provider-list">
              <FormSection
                v-for="(provider, index) in registerProviders"
                :key="providerKey(provider, index)"
                class="register-provider-card"
                surface="background"
                density="normal"
              >
                <div class="register-provider-head">
                  <div class="min-w-0">
                    <div class="register-provider-title">
                      <span>{{ providerTitle(provider, index) }}</span>
                      <MetaChip size="xs" tone="muted">{{ providerTypeLabel(providerType(provider)) }}</MetaChip>
                      <MetaChip v-if="provider.enable === false" size="xs" tone="warning">Dezactivat</MetaChip>
                      <MetaChip v-else-if="providerRequirementMessages(provider).length" size="xs" tone="danger">
                        Lipsă {{ providerRequirementMessages(provider).length }} elem.
                      </MetaChip>
                      <MetaChip v-else size="xs" tone="success">Poate porni</MetaChip>
                    </div>
                  </div>
                  <div class="register-provider-actions">
                    <Checkbox v-model="provider.enable" :disabled="registerConfig.enabled">
                      Activează
                    </Checkbox>
                    <Button
                      size="sm"
                      variant="ghost"
                      :disabled="registerConfig.enabled || registerProviders.length <= 1"
                      @click="deleteProvider(index)"
                    >
                      Șterge
                    </Button>
                  </div>
                </div>

                <SurfaceBox
                  v-if="provider.enable !== false && providerRequirementMessages(provider).length"
                  class="register-provider-message"
                  tone="danger"
                  density="compact"
                >
                  Lipsesc: {{ providerRequirementMessages(provider).join(', ') }}
                </SurfaceBox>

                <div class="register-provider-section">
                  <div class="register-provider-section-title">Configurație de bază</div>
                  <div class="register-form-grid register-form-grid--two">
                    <label class="register-field">
                      <span class="register-label">Tip</span>
                      <GroupedSelectMenu
                        :model-value="provider.type || 'cloudmail_gen'"
                        :groups="providerTypeGroups"
                        selected-indicator="none"
                        :disabled="registerConfig.enabled"
                        block
                        @update:model-value="value => updateProviderType(index, String(value))"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'gptmail'" class="register-field">
                      <span class="register-label">Sursă Key</span>
                      <GroupedSelectMenu
                        v-model="provider.key_mode"
                        :groups="gptMailKeyModeGroups"
                        selected-indicator="none"
                        :disabled="registerConfig.enabled"
                        block
                      />
                    </label>

                    <label v-if="providerUsesApiBase(provider)" class="register-field">
                      <span class="register-label">{{ apiBaseLabel(provider) }}</span>
                      <Input
                        v-model.trim="provider.api_base"
                        block
                        root-class="font-mono"
                        :disabled="registerConfig.enabled"
                        :placeholder="apiBasePlaceholder(provider)"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'cloudmail_gen'" class="register-field">
                      <span class="register-label">Email administrator</span>
                      <Input v-model.trim="provider.admin_email" block :disabled="registerConfig.enabled" />
                    </label>

                    <label v-if="providerUsesAdminPassword(provider)" class="register-field">
                      <span class="register-label">{{ providerType(provider) === 'ddg_mail' ? 'CF Admin Password' : 'Admin Password' }}</span>
                      <Input
                        v-model.trim="provider.admin_password"
                        block
                        root-class="font-mono"
                        :disabled="registerConfig.enabled"
                      />
                    </label>

                    <label v-if="providerUsesApiKey(provider) && !providerUsesPublicGptMailKey(provider)" class="register-field">
                      <span class="register-label">API Key</span>
                      <Input
                        v-model.trim="provider.api_key"
                        block
                        root-class="font-mono"
                        :disabled="registerConfig.enabled"
                      />
                    </label>

                    <label v-if="providerUsesDefaultDomain(provider)" class="register-field">
                      <span class="register-label">Domeniu implicit</span>
                      <Input
                        v-model.trim="provider.default_domain"
                        block
                        :placeholder="providerType(provider) === 'duckmail' ? 'duckmail.sbs' : providerType(provider) === 'gptmail' ? 'sk-ai.eu.cc' : ''"
                        :disabled="registerConfig.enabled"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'cloudmail_gen'" class="register-field">
                      <span class="register-label">Prefix email</span>
                      <Input
                        v-model.trim="provider.email_prefix"
                        block
                        :disabled="registerConfig.enabled"
                        placeholder="Opțional"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'moemail'" class="register-field">
                      <span class="register-label">Timp expirare</span>
                      <Input
                        v-model.number="provider.expiry_time"
                        type="number"
                        min="0"
                        block
                        :disabled="registerConfig.enabled"
                        placeholder="0 = implicit serviciu"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'ddg_mail'" class="register-field">
                      <span class="register-label">DDG Token</span>
                      <Input
                        v-model.trim="provider.ddg_token"
                        block
                        root-class="font-mono"
                        :disabled="registerConfig.enabled"
                        placeholder="DuckDuckGo Email Protection Bearer Token"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'ddg_mail'" class="register-field">
                      <span class="register-label">CF Inbox JWT</span>
                      <Input
                        v-model.trim="provider.cf_inbox_jwt"
                        block
                        root-class="font-mono"
                        :disabled="registerConfig.enabled"
                        placeholder="JWT inbox fix"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'ddg_mail'" class="register-field">
                      <span class="register-label">CF API Key</span>
                      <Input
                        v-model.trim="provider.cf_api_key"
                        block
                        root-class="font-mono"
                        :disabled="registerConfig.enabled"
                        placeholder="Opțional"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'ddg_mail'" class="register-field">
                      <span class="register-label">Metodă autentificare CF</span>
                      <GroupedSelectMenu
                        v-model="provider.cf_auth_mode"
                        :groups="cfAuthModeGroups"
                        selected-indicator="none"
                        :disabled="registerConfig.enabled"
                        block
                      />
                    </label>

                    <label v-if="providerType(provider) === 'ddg_mail'" class="register-field">
                      <span class="register-label">Cale creare</span>
                      <Input
                        v-model.trim="provider.cf_create_path"
                        block
                        root-class="font-mono"
                        :disabled="registerConfig.enabled"
                        placeholder="/api/new_address"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'ddg_mail'" class="register-field">
                      <span class="register-label">Cale listă emailuri</span>
                      <Input
                        v-model.trim="provider.cf_messages_path"
                        block
                        root-class="font-mono"
                        :disabled="registerConfig.enabled"
                        placeholder="/api/mails"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'yyds_mail'" class="register-field">
                      <span class="register-label">Subdomeniu</span>
                      <Input
                        :model-value="stringValue(provider.subdomain)"
                        block
                        :disabled="registerConfig.enabled"
                        @update:model-value="value => updateProviderField(index, 'subdomain', String(value || ''))"
                      />
                    </label>

                    <label v-if="providerType(provider) === 'inbucket'" class="register-checkbox-field">
                      <Checkbox v-model="provider.random_subdomain" :disabled="registerConfig.enabled">
                        Subdomeniu aleatoriu
                      </Checkbox>
                    </label>

                    <label v-if="providerType(provider) === 'yyds_mail'" class="register-checkbox-field">
                      <Checkbox v-model="provider.wildcard" :disabled="registerConfig.enabled">
                        Wildcard
                      </Checkbox>
                    </label>

                    <label v-if="providerType(provider) === 'gptmail'" class="register-checkbox-field register-checkbox-field--compact register-field--full">
                      <Checkbox v-model="provider.local_compose" :disabled="registerConfig.enabled">
                        Concatenare locală domenii cunoscute
                      </Checkbox>
                    </label>
                  </div>
                </div>

                <div v-if="providerType(provider) === 'gptmail'" class="register-provider-section register-provider-section--soft">
                  <div class="register-provider-section-title">Cotă GPTMail</div>
                  <div class="register-gptmail-panel">
                    <div class="register-gptmail-summary">
                      <MetaChip size="xs" :tone="gptMailStatusTone(index)">
                        {{ gptMailStatusTitle(index, provider) }}
                      </MetaChip>
                      <MetaChip size="xs" tone="muted">Key {{ gptMailKeyModeLabel(provider) }}</MetaChip>
                      <MetaChip v-if="gptMailStatusByIndex(index)?.key_hint" size="xs" tone="muted">
                        {{ gptMailStatusByIndex(index)?.key_hint }}
                      </MetaChip>
                      <MetaChip v-if="gptMailRemainingText(index)" size="xs" tone="info">
                        Rămas {{ gptMailRemainingText(index) }}
                      </MetaChip>
                      <MetaChip v-if="gptMailResetText(index)" size="xs" tone="muted">
                        {{ gptMailResetText(index) }}
                      </MetaChip>
                    </div>
                    <div class="register-provider-actions register-provider-actions--left">
                      <Button
                        size="xs"
                        variant="outline"
                        :disabled="registerConfig.enabled || gptMailStatusBusy(index)"
                        @click="checkGptMailStatus(index, provider)"
                      >
                        {{ gptMailStatusBusy(index) ? 'Se verifică' : 'Verifică cota' }}
                      </Button>
                    </div>
                    <p class="register-preview-line">{{ gptMailStatusHint(index, provider) }}</p>
                  </div>
                </div>

                <div
                  v-if="providerUsesDomainList(provider) || providerType(provider) === 'cloudmail_gen'"
                  class="register-provider-section"
                >
                  <div class="register-provider-section-title">Configurație domeniu</div>
                  <div class="register-provider-stack">
                    <label v-if="providerUsesDomainList(provider)" class="register-field">
                      <span class="register-label">{{ domainLabel(provider) }}</span>
                      <textarea
                        class="register-textarea"
                        :disabled="registerConfig.enabled"
                        :placeholder="domainPlaceholder(provider)"
                        :value="arrayText(provider.domain)"
                        @input="updateProviderArray(index, 'domain', $event)"
                      ></textarea>
                    </label>

                    <label v-if="providerType(provider) === 'cloudmail_gen'" class="register-field">
                      <span class="register-label">Prefix subdomeniu</span>
                      <textarea
                        class="register-textarea"
                        :disabled="registerConfig.enabled"
                        placeholder="Câte un prefix subdomeniu pe linie, gol = domeniu principal"
                        :value="arrayText(provider.subdomain)"
                        @input="updateProviderArray(index, 'subdomain', $event)"
                      ></textarea>
                    </label>
                  </div>
                </div>

                <div v-if="providerType(provider) === 'outlook_token'" class="register-provider-section register-provider-section--soft">
                  <div class="register-provider-section-title">Pool emailuri Outlook</div>

                  <div class="register-form-grid register-form-grid--three">
                    <label class="register-field">
                      <span class="register-label">Metodă citire</span>
                      <GroupedSelectMenu
                        v-model="provider.mode"
                        :groups="outlookModeGroups"
                        selected-indicator="none"
                        :disabled="registerConfig.enabled"
                        block
                      />
                    </label>

                    <label v-if="provider.mode !== 'graph'" class="register-field">
                      <span class="register-label">IMAP Host</span>
                      <Input
                        v-model.trim="provider.imap_host"
                        block
                        root-class="font-mono"
                        :disabled="registerConfig.enabled"
                        placeholder="outlook.office365.com"
                      />
                    </label>

                    <label class="register-field">
                      <span class="register-label">Număr emailuri citite</span>
                      <Input
                        v-model.number="provider.message_limit"
                        type="number"
                        min="1"
                        block
                        :disabled="registerConfig.enabled"
                      />
                    </label>
                  </div>

                  <div class="register-provider-section register-provider-section--soft">
                    <div class="register-provider-section-title">Alias plus</div>
                    <div class="register-form-grid register-form-grid--three">
                      <label class="register-checkbox-field register-checkbox-field--compact register-field--full">
                        <Checkbox v-model="provider.alias_enabled" :disabled="registerConfig.enabled">
                          Activează alias plus Outlook / Hotmail
                        </Checkbox>
                      </label>

                      <label class="register-field">
                        <span class="register-label">Aliasuri per email</span>
                        <Input
                          v-model.number="provider.alias_per_email"
                          type="number"
                          min="0"
                          max="200"
                          block
                          :disabled="registerConfig.enabled || !provider.alias_enabled"
                        />
                      </label>

                      <label class="register-field">
                        <span class="register-label">Prefix alias</span>
                        <Input
                          v-model.trim="provider.alias_prefix"
                          block
                          root-class="font-mono"
                          placeholder="c2api"
                          :disabled="registerConfig.enabled || !provider.alias_enabled"
                        />
                      </label>

                      <label class="register-checkbox-field register-checkbox-field--compact">
                        <Checkbox v-model="provider.alias_include_original" :disabled="registerConfig.enabled || !provider.alias_enabled">
                          Include email original
                        </Checkbox>
                      </label>
                    </div>
                    <p class="register-preview-line">{{ outlookAliasHint(provider) }}</p>
                  </div>

                  <label class="register-field">
                    <span class="register-label">Import pool emailuri</span>
                    <textarea
                      class="register-textarea register-textarea--tall"
                      :disabled="registerConfig.enabled"
                      :value="String(provider.mailboxes || '')"
                      placeholder="Câte unul pe linie: email----parolă----client_id----refresh_token"
                      @input="updateProviderField(index, 'mailboxes', ($event.target as HTMLTextAreaElement).value)"
                    ></textarea>
                  </label>

                  <div class="register-outlook-toolbar">
                    <div class="register-outlook-summary">
                      <MetaChip size="xs" tone="success">Disponibil {{ outlookPoolSummary(provider).available }}</MetaChip>
                      <MetaChip size="xs" tone="muted">Folosit {{ outlookPoolSummary(provider).used }}</MetaChip>
                      <MetaChip size="xs" :tone="outlookPoolSummary(provider).abnormal ? 'warning' : 'success'">
                        Anomalii {{ outlookPoolSummary(provider).abnormal }}
                      </MetaChip>
                      <MetaChip v-if="outlookPoolSummary(provider).pending" size="xs" tone="info">
                        De salvat {{ outlookPoolSummary(provider).pending }}
                      </MetaChip>
                    </div>

                    <FloatingActionMenu
                      label="Mai multe opțiuni"
                      :items="outlookPoolActionItems"
                      :disabled="registerConfig.enabled || legacySaving"
                      align="right"
                      placement="auto"
                      :trigger-min-width="96"
                      @select="handleOutlookPoolAction"
                    />
                  </div>

                  <p class="register-preview-line">{{ outlookPoolHint(provider) }}</p>
                  <details class="register-outlook-details">
                    <summary>Detalii pool emailuri</summary>
                    <div class="register-outlook-detail-chips">
                      <MetaChip size="xs" tone="muted">Salvat {{ outlookPoolSummary(provider).saved }}</MetaChip>
                      <MetaChip size="xs" tone="info">De salvat {{ outlookPoolSummary(provider).pending }}</MetaChip>
                      <MetaChip size="xs" tone="muted">Ocupat {{ outlookPoolSummary(provider).inUse }}</MetaChip>
                      <MetaChip size="xs" tone="warning">Necesită autentificare {{ outlookPoolSummary(provider).loginRequired }}</MetaChip>
                      <MetaChip size="xs" tone="warning">Invalid {{ outlookPoolSummary(provider).tokenInvalid }}</MetaChip>
                      <MetaChip size="xs" tone="danger">Eșuat {{ outlookPoolSummary(provider).failed }}</MetaChip>
                    </div>
                  </details>
                </div>
              </FormSection>
            </div>
          </FormSection>
        </div>

        <aside class="register-runtime-column">
          <FormSection title="Control execuție" density="roomy" class="register-runtime-section">
            <MetricStrip
              :items="registerMetricItems"
              columns-class="grid-cols-2 md:grid-cols-4"
              density="compact"
            />

            <div class="register-runtime-actions">
              <Button
                block
                variant="primary"
                :disabled="registerActionDisabled"
                @click="toggleLegacyTask"
              >
                {{ registerConfig.enabled ? 'Oprește' : 'Pornește' }}
              </Button>
              <Button
                block
                variant="outline"
                :disabled="legacySaving || !registerConfig || registerConfig.enabled"
                @click="resetLegacyStats"
              >
                Resetează
              </Button>
            </div>

            <SurfaceBox tone="muted" density="compact">
              {{ registerRuntimeHint }}
            </SurfaceBox>

            <SurfaceBox tone="muted" density="compact" class="register-runtime-tips">
              <p>Interceptare Cloudflare: activează FlareSolverr în setări și verifică containerele.</p>
              <p>Erorile HTTP 400 la înregistrare țin de风控 domeniu email. Încearcă alt domeniu.</p>
            </SurfaceBox>
          </FormSection>

          <RuntimeLogPanel
            class="register-runtime-log"
            title="Jurnal timp real"
            :lines="runtimeLogLines"
            :empty-title="'Niciun jurnal'"
            min-height="20rem"
            max-height="min(58vh, 38rem)"
          />
        </aside>
      </div>
    </PagePanel>
  </div>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import { Button, Checkbox, Input } from 'nanocat-ui'
import type { ActionMenuItem } from 'nanocat-ui'
import { proxyApi } from '@/api'
import { getAuthToken } from '@/api/client'
import { parseProxyReference, serializeProxyReference, type ProxyGroup } from '@/api/proxy'
import { registerApi, type GptMailStatus, type LegacyRegisterConfig, type OutlookMailboxParseStats, type RegisterProvider } from '@/api/register'
import { FloatingActionMenu, FormSection, MetaChip, MetricStrip, PageLoadingState, PagePanel, PanelHeader, RuntimeLogPanel, StateBadge, StateBlock, SurfaceBox, type RuntimeLogPanelLine } from '@/components/ai'
import GroupedSelectMenu from '@/components/ui/GroupedSelectMenu.vue'
import { useConfirmDialog } from '@/composables/useConfirmDialog'
import { useToast } from '@/composables/useToast'

type RegisterMode = 'total' | 'quota' | 'available'
type OutlookResetScope = 'all' | 'failed' | 'unused'
type RegisterProxyMode = 'global' | 'direct' | 'group' | 'custom'
type GptMailStatusState = {
  loading: boolean
  error: string
  data: GptMailStatus | null
}
type GptMailCheckOptions = {
  silent?: boolean
  force?: boolean
  reschedule?: boolean
}

const toast = useToast()
const confirmDialog = useConfirmDialog()

const legacyLoading = ref(false)
const legacySaving = ref(false)
const pollTimer = ref<number | null>(null)
const eventSource = ref<EventSource | null>(null)
const proxyGroups = ref<ProxyGroup[]>([])
const registerProxyMode = ref<RegisterProxyMode>('global')
const selectedRegisterProxyGroupId = ref('')
const customRegisterProxyInput = ref('')
const gptMailStatusStates = ref<Record<number, GptMailStatusState>>({})
const gptMailClockNow = ref(Date.now())
const gptMailRefreshTimers = new Map<number, number[]>()
const gptMailClockTimer = ref<number | null>(null)
const gptMailResetFallbackSeconds = 5 * 60

const defaultRegisterConfig: LegacyRegisterConfig = {
  mail: {
    request_timeout: 30,
    wait_timeout: 30,
    wait_interval: 2,
    user_agent: '',
    providers: [],
  },
  proxy: '',
  total: 10,
  threads: 3,
  mode: 'total',
  target_quota: 100,
  target_available: 10,
  check_interval: 5,
  enabled: false,
  stats: {
    success: 0,
    fail: 0,
    done: 0,
    running: 0,
    threads: 3,
    elapsed_seconds: 0,
    avg_seconds: 0,
    success_rate: 0,
    current_quota: 0,
    current_available: 0,
  },
  logs: [],
}

const registerConfig = ref<LegacyRegisterConfig | null>(null)

const registerModeOptions = [
  { value: 'total', label: 'Înregistrare după număr' },
  { value: 'quota', label: 'Oprește la cotă' },
  { value: 'available', label: 'Oprește la număr conturi' },
]
const registerModeGroups = [{ options: registerModeOptions }]
const registerProxyModeOptions = [
  { value: 'global', label: 'Folosește proxy implicit' },
  { value: 'direct', label: 'Conexiune directă' },
  { value: 'group', label: 'Grup proxy' },
  { value: 'custom', label: 'Proxy personalizat' },
]
const registerProxyModeGroups = [{ options: registerProxyModeOptions }]

const providerTypeOptions = [
  { value: 'cloudmail_gen', label: 'CloudMail Gen' },
  { value: 'cloudflare_temp_email', label: 'Cloudflare Temp Email' },
  { value: 'tempmail_lol', label: 'TempMail.lol' },
  { value: 'moemail', label: 'MoEmail' },
  { value: 'inbucket', label: 'Inbucket' },
  { value: 'duckmail', label: 'DuckMail' },
  { value: 'gptmail', label: 'GPTMail' },
  { value: 'yyds_mail', label: 'YYDS Mail' },
  { value: 'ddg_mail', label: 'DDG + CF Inbox' },
  { value: 'outlook_token', label: 'Pool credențiale Microsoft' },
]
const providerTypeGroups = [{ options: providerTypeOptions }]

const cfAuthModeOptions = [
  { value: 'none', label: 'Nu atașa' },
  { value: 'bearer', label: 'Bearer' },
  { value: 'x-api-key', label: 'X-API-Key' },
  { value: 'query-key', label: 'Query key' },
]
const cfAuthModeGroups = [{ options: cfAuthModeOptions }]
const gptMailKeyModeOptions = [
  { value: 'public', label: 'Key test public' },
  { value: 'custom', label: 'Key personalizat' },
]
const gptMailKeyModeGroups = [{ options: gptMailKeyModeOptions }]

const outlookModeOptions = [
  { value: 'graph', label: 'Graph API' },
  { value: 'imap', label: 'IMAP' },
  { value: 'auto', label: 'Fallback automat' },
]
const outlookModeGroups = [{ options: outlookModeOptions }]
const outlookPoolActionItems: ActionMenuItem[] = [
  { key: 'retry_failed', label: 'Reîncearcă emailuri anormale' },
  { key: 'failed', label: 'Eliberează doar stări anormale' },
  { key: 'unused', label: 'Șterge materiale nefolosite', danger: true, dividerBefore: true },
  { key: 'all', label: 'Resetează stare pool emailuri', danger: true },
]
const providerCommonKeys = ['id', 'enable', 'type', 'label'] as const
const providerTypeKeys: Record<string, string[]> = {
  cloudmail_gen: ['api_base', 'admin_email', 'admin_password', 'domain', 'subdomain', 'email_prefix'],
  cloudflare_temp_email: ['api_base', 'admin_password', 'domain'],
  tempmail_lol: ['api_key', 'domain'],
  moemail: ['api_base', 'api_key', 'domain', 'expiry_time'],
  inbucket: ['api_base', 'domain', 'random_subdomain'],
  duckmail: ['api_key', 'default_domain'],
  gptmail: ['key_mode', 'api_key', 'default_domain', 'local_compose'],
  yyds_mail: ['api_base', 'api_key', 'domain', 'subdomain', 'wildcard'],
  ddg_mail: ['api_base', 'ddg_token', 'cf_inbox_jwt', 'admin_password', 'cf_api_key', 'cf_auth_mode', 'cf_create_path', 'cf_messages_path'],
  outlook_token: ['mailboxes', 'mode', 'imap_host', 'message_limit', 'alias_enabled', 'alias_per_email', 'alias_prefix', 'alias_include_original'],
}
const providerLocalOnlyKeys: Record<string, string[]> = {
  outlook_token: ['mailboxes_count', 'mailboxes_base_count', 'mailboxes_alias_count', 'mailboxes_preview', 'mailboxes_stats', 'mailboxes_parse_stats'],
}

const registerProviders = computed(() => registerConfig.value?.mail.providers || [])
const registerProxyGroupOptions = computed(() => {
  const rows = proxyGroups.value.map((group) => ({
    label: `${group.enabled === false ? 'Dezactivat · ' : ''}${group.name || group.id}${Array.isArray(group.nodes) ? ` · ${group.nodes.length} noduri` : ''}`,
    value: group.id,
  }))
  const selectedId = selectedRegisterProxyGroupId.value
  if (selectedId && !rows.some((item) => item.value === selectedId)) {
    rows.unshift({ label: `Grup necunoscut · ${selectedId}`, value: selectedId })
  }
  return [
    { label: 'Selectează grup proxy', value: '' },
    ...rows,
  ]
})
const registerProxyGroupGroups = computed(() => [{ options: registerProxyGroupOptions.value }])
const registerProxyHint = computed(() => {
  if (registerProxyMode.value === 'direct') return 'Sarcina forțează conexiune directă, ignoră proxy implicit.'
  if (registerProxyMode.value === 'group') return 'Sarcina folosește grupul selectat; nu revine la proxy implicit.'
  if (registerProxyMode.value === 'custom') return 'Doar această sarcină folosește adresa proxy.'
  return 'Folosește proxy implicit din setări; fără proxy dacă e direct.'
})
const enabledProviderCount = computed(() => registerProviders.value.filter(provider => provider.enable !== false).length)
const enabledProviderIssueCount = computed(() =>
  registerProviders.value
    .filter(provider => provider.enable !== false)
    .reduce((total, provider) => total + providerRequirementMessages(provider).length, 0),
)
const registerActionDisabled = computed(() => {
  if (legacySaving.value || !registerConfig.value) return true
  if (registerConfig.value.enabled) return false
  return enabledProviderCount.value === 0 || enabledProviderIssueCount.value > 0
})
const legacyStats = computed(() => ({ ...defaultRegisterConfig.stats, ...(registerConfig.value?.stats || {}) }))
const legacyLogs = computed(() => [...(registerConfig.value?.logs || [])])
const registerRuntimeHint = computed(() => {
  if (enabledProviderCount.value === 0) return 'Activează cel puțin o sursă email.'
  if (enabledProviderIssueCount.value > 0) return `Mai sunt ${enabledProviderIssueCount.value} configurări obligatorii neterminate.`
  if (registerConfig.value?.enabled) return 'Sarcină activă, configurație blocată.'
  return 'Configurația se salvează automat înainte de pornire.'
})

const registerMetricItems = computed(() => {
  const stats = legacyStats.value
  return [
    { key: 'success', label: 'Succes', value: stats.success || 0, meta: `Rată succes ${stats.success_rate || 0}%` },
    { key: 'fail', label: 'Eșec', value: stats.fail || 0 },
    { key: 'done', label: 'Finalizat', value: stats.done || 0 },
    { key: 'running', label: 'Activ / Fire', value: `${stats.running || 0} / ${stats.threads || registerConfig.value?.threads || 0}` },
    { key: 'elapsed', label: 'Timp rulare', value: `${stats.elapsed_seconds || 0}s` },
    { key: 'avg', label: 'Timp mediu', value: `${stats.avg_seconds || 0}s` },
    { key: 'quota', label: 'Cotă curentă', value: stats.current_quota || 0 },
    { key: 'available', label: 'Conturi normale', value: stats.current_available || 0 },
  ]
})

const runtimeLogLines = computed<RuntimeLogPanelLine[]>(() => legacyLogs.value.slice().reverse().map((item, index) => ({
  key: `${item.time || 'log'}-${index}`,
  time: formatClock(item.time),
  text: item.text || '-',
  level: normalizeLogLevel(item.level),
})))

function normalizeRegisterConfig(raw: LegacyRegisterConfig): LegacyRegisterConfig {
  const mail = {
    ...defaultRegisterConfig.mail,
    ...(raw.mail || {}),
    providers: Array.isArray(raw.mail?.providers) ? raw.mail.providers.map(item => normalizeProvider(item)) : [],
  }
  if (!mail.providers.length) {
    mail.providers = [defaultProvider()]
  }
  return {
    ...defaultRegisterConfig,
    ...raw,
    mail,
    stats: { ...defaultRegisterConfig.stats, ...(raw.stats || {}) },
    logs: Array.isArray(raw.logs) ? raw.logs : [],
  }
}

function normalizeProvider(provider: RegisterProvider): RegisterProvider {
  const type = providerType(provider)
  const normalized = {
    ...defaultProvider(type),
    ...provider,
    id: String(provider.id || provider.provider_id || '').trim() || createProviderId(type),
    type,
    enable: provider.enable !== false,
  }
  if (type === 'gptmail' && !provider.key_mode && isFilled(provider.api_key)) {
    normalized.key_mode = 'custom'
  }
  return normalized
}

function defaultProvider(type = 'cloudmail_gen'): RegisterProvider {
  const base = { id: createProviderId(type), enable: true, type }
  switch (type) {
    case 'cloudmail_gen':
      return { ...base, api_base: '', admin_email: '', admin_password: '', domain: [], subdomain: [], email_prefix: '' }
    case 'cloudflare_temp_email':
      return { ...base, api_base: '', admin_password: '', domain: [] }
    case 'tempmail_lol':
      return { ...base, api_key: '', domain: [] }
    case 'moemail':
      return { ...base, api_base: '', api_key: '', domain: [], expiry_time: 0 }
    case 'inbucket':
      return { ...base, api_base: '', domain: [], random_subdomain: true }
    case 'duckmail':
      return { ...base, api_key: '', default_domain: 'duckmail.sbs' }
    case 'gptmail':
      return { ...base, key_mode: 'public', api_key: '', default_domain: '', local_compose: false }
    case 'yyds_mail':
      return { ...base, api_base: 'https://maliapi.215.im/v1', api_key: '', domain: [], subdomain: '', wildcard: false }
    case 'ddg_mail':
      return {
        ...base,
        api_base: '',
        ddg_token: '',
        cf_inbox_jwt: '',
        admin_password: '',
        cf_api_key: '',
        cf_auth_mode: 'none',
        cf_create_path: '/api/new_address',
        cf_messages_path: '/api/mails',
      }
    case 'outlook_token':
      return {
        ...base,
        mailboxes: '',
        mode: 'auto',
        imap_host: 'outlook.office365.com',
        message_limit: 10,
        alias_enabled: false,
        alias_per_email: 5,
        alias_prefix: 'c2api',
        alias_include_original: true,
      }
    default:
      return base
  }
}

function providerType(provider: RegisterProvider) {
  return String(provider.type || 'cloudmail_gen')
}

function createProviderId(type = 'provider') {
  const suffix = typeof crypto !== 'undefined' && typeof crypto.randomUUID === 'function'
    ? crypto.randomUUID().replace(/-/g, '').slice(0, 12)
    : Math.random().toString(36).slice(2, 14).padEnd(12, '0')
  return `${type}-${suffix}`
}

function providerKey(provider: RegisterProvider, index: number) {
  return String(provider.id || provider.provider_id || '').trim() || `${providerType(provider)}-${index}`
}

function providerTitle(provider: RegisterProvider, index: number) {
  return `Sursă email ${index + 1}`
}

function providerTypeLabel(type: string) {
  return providerTypeOptions.find(item => item.value === type)?.label || type
}

function providerKeysForType(type: string, includeLocalOnly = false) {
  return [
    ...providerCommonKeys,
    ...(providerTypeKeys[type] || []),
    ...(includeLocalOnly ? providerLocalOnlyKeys[type] || [] : []),
  ]
}

function providerHasKnownType(type: string) {
  return Object.prototype.hasOwnProperty.call(providerTypeKeys, type)
}

function listFromDraft(value: unknown) {
  if (Array.isArray(value)) return value.map(String).map(item => item.trim()).filter(Boolean)
  return String(value || '')
    .split(/[\n,]/)
    .map(item => item.trim())
    .filter(Boolean)
}

function providerDraftValue(type: string, key: string, value: unknown) {
  if (key === 'domain') return listFromDraft(value)
  if (key === 'subdomain') {
    if (type === 'cloudmail_gen') return listFromDraft(value)
    if (type === 'yyds_mail') return Array.isArray(value) ? value.join('\n') : String(value || '')
  }
  return value
}

function providerWithTypeDraft(current: RegisterProvider, type: string): RegisterProvider {
  const defaults = defaultProvider(type)
  const next: RegisterProvider = {
    ...current,
    ...defaults,
    id: String(current.id || current.provider_id || defaults.id || '').trim(),
    type,
    enable: current.enable !== false,
  }

  for (const key of providerKeysForType(type, true)) {
    if (key === 'type' || key === 'enable') continue
    if (current[key] !== undefined) {
      next[key] = providerDraftValue(type, key, current[key])
    }
  }

  next.type = type
  next.enable = current.enable !== false

  return next
}

function isFilled(value: unknown) {
  return String(value ?? '').trim().length > 0
}

function listHasValue(value: unknown) {
  if (Array.isArray(value)) return value.some(item => isFilled(item))
  return isFilled(value)
}

function providerRequirementMessages(provider: RegisterProvider) {
  const type = providerType(provider)
  const missing: string[] = []
  const requireValue = (value: unknown, label: string) => {
    if (!isFilled(value)) missing.push(label)
  }
  const requireList = (value: unknown, label: string) => {
    if (!listHasValue(value)) missing.push(label)
  }

  switch (type) {
    case 'cloudmail_gen':
      requireValue(provider.api_base, 'CloudMail URL')
      requireValue(provider.admin_email, 'Email administrator')
      requireValue(provider.admin_password, 'Admin Password')
      requireList(provider.domain, 'Domeniu email')
      break
    case 'cloudflare_temp_email':
      requireValue(provider.api_base, 'API Base')
      requireValue(provider.admin_password, 'Admin Password')
      requireList(provider.domain, 'Domeniu')
      break
    case 'moemail':
      requireValue(provider.api_base, 'API Base')
      requireValue(provider.api_key, 'API Key')
      requireList(provider.domain, 'Domeniu')
      break
    case 'inbucket':
      requireValue(provider.api_base, 'API Base')
      requireList(provider.domain, 'Domeniu de bază')
      break
    case 'duckmail':
      requireValue(provider.api_key, 'API Key')
      break
    case 'gptmail':
      if (!providerUsesPublicGptMailKey(provider)) requireValue(provider.api_key, 'API Key')
      if (provider.local_compose) requireValue(provider.default_domain, 'Domeniu implicit')
      break
    case 'yyds_mail':
      requireValue(provider.api_key, 'API Key')
      break
    case 'ddg_mail':
      requireValue(provider.api_base, 'CF API Base')
      requireValue(provider.ddg_token, 'DDG Token')
      requireValue(provider.cf_inbox_jwt, 'CF Inbox JWT')
      break
    case 'outlook_token': {
      const savedCount = Number(provider.mailboxes_count || 0)
      if (savedCount <= 0 && pendingOutlookCount(provider) <= 0) missing.push('Pool credențiale Microsoft')
      break
    }
    default:
      break
  }

  return missing
}

function updateProviderType(index: number, type: string) {
  if (!registerConfig.value) return
  clearGptMailState(index)
  const providers = [...registerProviders.value]
  const current = providers[index] || {}
  providers[index] = providerWithTypeDraft(current, type)
  registerConfig.value.mail.providers = providers
}

function updateProviderField(index: number, key: string, value: unknown) {
  const provider = registerProviders.value[index]
  if (!provider) return
  provider[key] = value
}

function providerUsesApiBase(provider: RegisterProvider) {
  return ['cloudmail_gen', 'cloudflare_temp_email', 'moemail', 'inbucket', 'yyds_mail', 'ddg_mail'].includes(providerType(provider))
}

function providerUsesApiKey(provider: RegisterProvider) {
  return ['tempmail_lol', 'moemail', 'duckmail', 'gptmail', 'yyds_mail'].includes(providerType(provider))
}

function providerUsesPublicGptMailKey(provider: RegisterProvider) {
  return providerType(provider) === 'gptmail' && String(provider.key_mode || 'public') !== 'custom'
}

function providerUsesAdminPassword(provider: RegisterProvider) {
  return ['cloudmail_gen', 'cloudflare_temp_email', 'ddg_mail'].includes(providerType(provider))
}

function providerUsesDefaultDomain(provider: RegisterProvider) {
  return ['duckmail', 'gptmail'].includes(providerType(provider))
}

function providerUsesDomainList(provider: RegisterProvider) {
  return ['cloudmail_gen', 'tempmail_lol', 'cloudflare_temp_email', 'moemail', 'inbucket', 'yyds_mail'].includes(providerType(provider))
}

function apiBaseLabel(provider: RegisterProvider) {
  const type = providerType(provider)
  if (type === 'cloudmail_gen') return 'CloudMail URL'
  if (type === 'ddg_mail') return 'CF API Base'
  return 'API Base'
}

function apiBasePlaceholder(provider: RegisterProvider) {
  const type = providerType(provider)
  if (type === 'yyds_mail') return 'https://maliapi.215.im/v1'
  return ''
}

function domainLabel(provider: RegisterProvider) {
  const type = providerType(provider)
  if (type === 'inbucket') return 'Domeniu de bază'
  if (type === 'cloudmail_gen') return 'Domeniu email'
  return 'Domeniu'
}

function domainPlaceholder(provider: RegisterProvider) {
  const type = providerType(provider)
  if (type === 'inbucket') return 'Câte un domeniu de bază pe linie, cu subdomeniu aleatoriu'
  if (type === 'cloudmail_gen') return 'Câte un domeniu email pe linie'
  if (type === 'cloudflare_temp_email') return 'Câte un domeniu pe linie'
  if (type === 'moemail') return 'Câte un domeniu pe linie'
  if (type === 'tempmail_lol') return 'Câte un domeniu pe linie, gol = implicit serviciu'
  if (type === 'yyds_mail') return 'Câte un domeniu pe linie, opțional'
  return 'Câte un domeniu pe linie'
}

function gptMailKeyModeLabel(provider: RegisterProvider) {
  return providerUsesPublicGptMailKey(provider) ? 'Public' : 'Personalizat'
}

function outlookPoolStats(provider: RegisterProvider) {
  return provider.mailboxes_stats || {}
}

function numeric(value: unknown) {
  return Number(value || 0) || 0
}

function pendingOutlookCount(provider: RegisterProvider) {
  return String(provider.mailboxes || '')
    .split(/\r?\n/)
    .map(line => line.trim())
    .filter(line => line && line.split('----').length >= 4)
    .length
}

function outlookPoolSummary(provider: RegisterProvider) {
  const stats = outlookPoolStats(provider)
  const inUse = numeric(stats.in_use)
  const loginRequired = numeric(stats.login_required)
  const tokenInvalid = numeric(stats.token_invalid)
  const failed = numeric(stats.failed)

  return {
    saved: numeric(provider.mailboxes_count),
    pending: pendingOutlookCount(provider),
    available: numeric(stats.unused),
    used: numeric(stats.used),
    inUse,
    loginRequired,
    tokenInvalid,
    failed,
    abnormal: inUse + loginRequired + tokenInvalid + failed,
  }
}

function outlookAliasSummary(provider: RegisterProvider) {
  const base = numeric(provider.mailboxes_base_count || provider.mailboxes_count)
  const alias = numeric(provider.mailboxes_alias_count)
  const perEmail = numeric(provider.alias_per_email)
  const includeOriginal = provider.alias_include_original !== false
  const multiplier = provider.alias_enabled ? perEmail + (includeOriginal ? 1 : 0) : 1
  const pending = pendingOutlookCount(provider)
  return {
    enabled: Boolean(provider.alias_enabled),
    base,
    alias,
    perEmail,
    includeOriginal,
    multiplier,
    pending,
    pendingExpanded: provider.alias_enabled ? pending * multiplier : pending,
  }
}

function outlookAliasHint(provider: RegisterProvider) {
  const summary = outlookAliasSummary(provider)
  if (!summary.enabled) return 'Alias plus dezactivat, se folosește emailul importat direct.'
  if (summary.pending > 0) {
    return `După salvare, importul se extinde la ~${summary.pendingExpanded} adrese; autentificarea folosește credențialele originale.`
  }
  if (summary.base > 0) {
    return `Salvate ${summary.base} emailuri originale, generate ${summary.alias} aliasuri; autentificarea folosește credențialele originale.`
  }
  return 'După salvare se generează aliasuri plus pentru Outlook/Hotmail; autentificarea folosește credențialele originale.'
}

function outlookPoolHint(provider: RegisterProvider) {
  const summary = outlookPoolSummary(provider)
  if (summary.pending > 0) return `${summary.pending} de salvat, intră în pool după salvare.`
  if (summary.saved <= 0) return 'Niciun material Microsoft salvat încă.'
  if (summary.available <= 0 && summary.abnormal <= 0) return 'Stoc epuizat, importă materiale noi.'
  if (summary.abnormal > 0) return `${summary.abnormal} cu anomalii, eliberează sau reîncearcă din meniu.`
  return `Salvate ${summary.saved} materiale Microsoft.`
}

function gptMailState(index: number): GptMailStatusState {
  return gptMailStatusStates.value[index] || { loading: false, error: '', data: null }
}

function setGptMailState(index: number, state: GptMailStatusState) {
  gptMailStatusStates.value = { ...gptMailStatusStates.value, [index]: state }
}

function clearGptMailRefreshTimer(index: number) {
  const timers = gptMailRefreshTimers.get(index) || []
  timers.forEach(timer => window.clearTimeout(timer))
  if (timers.length) {
    gptMailRefreshTimers.delete(index)
  }
}

function clearAllGptMailRefreshTimers() {
  gptMailRefreshTimers.forEach(timers => timers.forEach(timer => window.clearTimeout(timer)))
  gptMailRefreshTimers.clear()
}

function clearGptMailState(index: number) {
  clearGptMailRefreshTimer(index)
  const next = { ...gptMailStatusStates.value }
  delete next[index]
  gptMailStatusStates.value = next
}

function pruneGptMailStates() {
  const next: Record<number, GptMailStatusState> = {}
  Object.entries(gptMailStatusStates.value).forEach(([key, state]) => {
    const index = Number(key)
    const provider = registerProviders.value[index]
    if (provider && providerType(provider) === 'gptmail') {
      next[index] = state
    } else {
      clearGptMailRefreshTimer(index)
    }
  })
  Array.from(gptMailRefreshTimers.keys()).forEach((index) => {
    const provider = registerProviders.value[index]
    if (!provider || providerType(provider) !== 'gptmail') clearGptMailRefreshTimer(index)
  })
  gptMailStatusStates.value = next
}

function gptMailSecondsUntilReset(status: GptMailStatus, now = gptMailClockNow.value) {
  const resetAt = Date.parse(String(status.reset_at || ''))
  if (Number.isFinite(resetAt)) {
    return Math.ceil((resetAt - now) / 1000)
  }
  const seconds = Number(status.seconds_until_reset)
  if (!Number.isFinite(seconds) || seconds <= 0) return null
  const checkedAt = Date.parse(String(status.checked_at || ''))
  if (Number.isFinite(checkedAt)) {
    return Math.ceil((checkedAt + seconds * 1000 - now) / 1000)
  }
  return Math.ceil(seconds)
}

function gptMailTimerDelay(seconds: number) {
  return Math.min(Math.max(seconds * 1000, 1000), 2_147_483_000)
}

function gptMailResetDelays(status: GptMailStatus) {
  const seconds = gptMailSecondsUntilReset(status, Date.now())
  if (seconds === null) return []
  const resetSeconds = Math.max(0, seconds)
  return [
    { delay: gptMailTimerDelay(resetSeconds), reschedule: false },
    { delay: gptMailTimerDelay(resetSeconds + gptMailResetFallbackSeconds), reschedule: true },
  ]
}

function scheduleGptMailRefresh(index: number, status: GptMailStatus) {
  clearGptMailRefreshTimer(index)
  if (String(status.key_mode || 'public') !== 'public') return
  const timers = gptMailResetDelays(status).map(({ delay, reschedule }) => {
    let timer = 0
    timer = window.setTimeout(() => {
      const activeTimers = gptMailRefreshTimers.get(index) || []
      const nextTimers = activeTimers.filter(item => item !== timer)
      if (nextTimers.length) {
        gptMailRefreshTimers.set(index, nextTimers)
      } else {
        gptMailRefreshTimers.delete(index)
      }
      const provider = registerProviders.value[index]
      if (!provider || providerType(provider) !== 'gptmail') return
      void refreshGptMailPublicKey(index, provider, { reschedule })
    }, delay)
    return timer
  })
  if (timers.length) gptMailRefreshTimers.set(index, timers)
}

function gptMailStatusByIndex(index: number) {
  return gptMailState(index).data
}

function gptMailStatusBusy(index: number) {
  return gptMailState(index).loading
}

function gptMailStatusTone(index: number) {
  const state = gptMailState(index)
  if (state.loading) return 'info'
  if (state.error) return 'danger'
  if (!state.data) return 'muted'
  if (state.data.is_active === false) return 'warning'
  return 'success'
}

function gptMailStatusTitle(index: number, provider: RegisterProvider) {
  const state = gptMailState(index)
  if (state.loading) return 'Se verifică'
  if (state.error) return 'Verificare eșuată'
  if (!state.data) return providerUsesPublicGptMailKey(provider) ? 'Key public' : 'Neverificat'
  return state.data.is_active === false ? 'Indisponibil' : 'Disponibil'
}

function formatGptMailNumber(value: unknown) {
  const number = Number(value)
  if (!Number.isFinite(number)) return ''
  if (number < 0) return 'Nelimitat'
  return new Intl.NumberFormat().format(number)
}

function formatGptMailDuration(seconds: unknown) {
  const total = Number(seconds)
  if (!Number.isFinite(total) || total <= 0) return ''
  if (total < 60) return `Resetare în ${Math.ceil(total)}s`
  const hours = Math.floor(total / 3600)
  const minutes = Math.floor((total % 3600) / 60)
  if (hours > 0) return `Resetare în ${hours}h ${minutes}m`
  return `Resetare în ${Math.max(1, minutes)}m`
}

function gptMailRemainingText(index: number) {
  const status = gptMailStatusByIndex(index)
  if (!status) return ''
  if (String(status.key_mode || '') === 'custom') {
    const remaining = formatGptMailNumber(status.remaining_total)
    const total = formatGptMailNumber(status.total_limit)
    if (remaining && total) return `${remaining} / ${total}`
    if (remaining) return remaining
  }
  return formatGptMailNumber(status.remaining_today ?? status.remaining_total)
}

function gptMailResetText(index: number) {
  const status = gptMailStatusByIndex(index)
  if (!status) return ''
  if (String(status.key_mode || '') === 'custom' && !status.reset_at && !status.seconds_until_reset) return ''
  const seconds = gptMailSecondsUntilReset(status)
  const countdown = formatGptMailDuration(seconds)
  if (countdown) return countdown
  if (seconds !== null && seconds <= 0) return 'Așteaptă reîmprospătare'
  if (status.reset_at) return `Resetare ${formatClock(status.reset_at)}`
  return ''
}

function gptMailStatusHint(index: number, provider: RegisterProvider) {
  const state = gptMailState(index)
  if (state.error) return state.error
  if (provider.local_compose && !String(provider.default_domain || '').trim()) {
    return 'Modul concatenare locală necesită domeniu implicit.'
  }
  if (provider.local_compose) {
    return 'Concatenarea locală economisește un apel API; verifică domeniul.'
  }
  if (!state.data) {
    return providerUsesPublicGptMailKey(provider)
      ? 'Key public GPTMail: backend-ul îl preia automat la pornire.'
      : 'Completează Key personalizat pentru a verifica cota.'
  }
  if (String(state.data.key_mode || provider.key_mode || '') === 'custom') {
    const totalUsed = formatGptMailNumber(state.data.total_usage)
    const totalLimit = formatGptMailNumber(state.data.total_limit)
    const totalRemaining = formatGptMailNumber(state.data.remaining_total)
    const checkedText = state.data.checked_at ? `Verificat la ${formatClock(state.data.checked_at)}` : 'Stare actualizată'
    const resetText = state.data.reset_at ? `Resetare ${formatClock(state.data.reset_at)}` : 'Key personalizat nu a returnat timp resetare'
    if (totalUsed && totalLimit) {
      return `Total folosit ${totalUsed} / ${totalLimit}${totalRemaining ? `, rămas ${totalRemaining}` : ''}, ${checkedText}; ${resetText}.`
    }
    if (totalRemaining) return `Total rămas ${totalRemaining}, ${checkedText}; ${resetText}.`
    return `${checkedText}; ${resetText}.`
  }
  const used = formatGptMailNumber(state.data.used_today)
  const limit = formatGptMailNumber(state.data.daily_limit)
  if (used && limit) return `Folosit azi ${used} / ${limit}, ${state.data.checked_at ? `verificat la ${formatClock(state.data.checked_at)}` : 'stare actualizată'}.`
  return state.data.checked_at ? `Stare actualizată, verificat la ${formatClock(state.data.checked_at)}.` : 'Stare actualizată.'
}

async function checkGptMailStatus(index: number, provider: RegisterProvider, options: GptMailCheckOptions = {}) {
  const previous = gptMailState(index).data
  setGptMailState(index, { ...gptMailState(index), loading: true, error: '' })
  try {
    const response = await registerApi.getGptMailStatus(sanitizeProvider(provider), options.force ?? true)
    setGptMailState(index, { loading: false, error: '', data: response.status })
    if (options.reschedule !== false) scheduleGptMailRefresh(index, response.status)
    if (!options.silent) toast.success('Cota GPTMail actualizată')
  } catch (error: any) {
    const message = error?.message || 'Verificare cotă GPTMail eșuată'
    setGptMailState(index, { loading: false, error: message, data: previous })
    if (!options.silent) toast.error(message)
  }
}

async function refreshGptMailPublicKey(index: number, provider: RegisterProvider, options: GptMailCheckOptions = {}) {
  const previous = gptMailState(index).data
  try {
    const response = await registerApi.refreshGptMailKey(sanitizeProvider(provider), options.force ?? true)
    setGptMailState(index, { loading: false, error: '', data: response.status })
    if (options.reschedule !== false) scheduleGptMailRefresh(index, response.status)
  } catch (error: any) {
    setGptMailState(index, { loading: false, error: error?.message || 'Reîmprospătare key eșuată', data: previous })
  }
}

function sanitizeProvider(provider: RegisterProvider): RegisterProvider {
  const copy = { ...provider }
  delete (copy as any).mailboxes_count
  delete (copy as any).mailboxes_base_count
  delete (copy as any).mailboxes_alias_count
  delete (copy as any).mailboxes_preview
  delete (copy as any).mailboxes_stats
  delete (copy as any).mailboxes_parse_stats
  return copy
}

function arrayText(arr: string[] | undefined): string {
  return (arr || []).join('\n')
}

function stringValue(value: unknown): string {
  return String(value ?? '')
}

function formatClock(time: string | undefined): string {
  if (!time) return '-'
  try {
    const d = new Date(time)
    return d.toLocaleTimeString('ro-RO', { hour: '2-digit', minute: '2-digit', second: '2-digit' })
  } catch {
    return time
  }
}

function normalizeLogLevel(level: string | undefined): string {
  const l = (level || 'info').toLowerCase()
  if (['error', 'err'].includes(l)) return 'error'
  if (['warn', 'warning'].includes(l)) return 'warning'
  if (['debug'].includes(l)) return 'debug'
  return 'info'
}

async function loadLegacyConfig() {
  legacyLoading.value = true
  try {
    const data = await registerApi.getLegacyConfig()
    registerConfig.value = normalizeRegisterConfig(data)
  } catch (error: any) {
    toast.error(error?.message || 'Eroare la încărcarea configurației')
  } finally {
    legacyLoading.value = false
  }
}

async function saveLegacyConfig() {
  if (!registerConfig.value) return
  legacySaving.value = true
  try {
    await registerApi.saveLegacyConfig(registerConfig.value)
    toast.success('Configurație salvată')
  } catch (error: any) {
    toast.error(error?.message || 'Eroare la salvare')
  } finally {
    legacySaving.value = false
  }
}

async function toggleLegacyTask() {
  if (!registerConfig.value) return
  const next = !registerConfig.value.enabled
  try {
    if (next) {
      await registerApi.startLegacyTask()
      toast.success('Sarcină pornită')
    } else {
      await registerApi.stopLegacyTask()
      toast.success('Sarcină oprită')
    }
    await loadLegacyConfig()
  } catch (error: any) {
    toast.error(error?.message || 'Eroare la comutare')
  }
}

async function resetLegacyStats() {
  if (!registerConfig.value) return
  const ok = await confirmDialog.confirm({
    title: 'Resetează statistici',
    message: 'Sigur resetezi toate statisticile de înregistrare?',
    confirmText: 'Resetează',
    cancelText: 'Anulează',
    tone: 'danger',
  })
  if (!ok) return
  try {
    await registerApi.resetLegacyStats()
    toast.success('Statistici resetate')
    await loadLegacyConfig()
  } catch (error: any) {
    toast.error(error?.message || 'Eroare la resetare')
  }
}

function addProvider() {
  if (!registerConfig.value) return
  const providers = [...registerProviders.value]
  providers.push(defaultProvider())
  registerConfig.value.mail.providers = providers
}

function deleteProvider(index: number) {
  if (!registerConfig.value) return
  const providers = [...registerProviders.value]
  providers.splice(index, 1)
  if (!providers.length) providers.push(defaultProvider())
  registerConfig.value.mail.providers = providers
  clearGptMailState(index)
}

function updateProviderArray(index: number, key: string, event: Event) {
  const provider = registerProviders.value[index]
  if (!provider) return
  const text = (event.target as HTMLTextAreaElement).value
  provider[key] = text.split('\n').map(s => s.trim()).filter(Boolean)
}

function setRegisterProxyMode(mode: RegisterProxyMode) {
  registerProxyMode.value = mode
  if (mode === 'group') {
    if (!selectedRegisterProxyGroupId.value && proxyGroups.value.length) {
      selectedRegisterProxyGroupId.value = proxyGroups.value[0].id
    }
  } else if (mode === 'custom') {
    if (!customRegisterProxyInput.value) {
      customRegisterProxyInput.value = registerConfig.value?.proxy || ''
    }
  }
}

function selectRegisterProxyGroup(groupId: string) {
  selectedRegisterProxyGroupId.value = groupId
}

function setCustomRegisterProxyInput(value: string) {
  customRegisterProxyInput.value = value
}

async function loadProxyGroups() {
  try {
    const data = await proxyApi.getProxyGroups()
    proxyGroups.value = data.groups || []
  } catch {
    proxyGroups.value = []
  }
}

function handleOutlookPoolAction(item: ActionMenuItem) {
  const scope = item.key as OutlookResetScope
  handleOutlookPoolReset(scope)
}

async function handleOutlookPoolReset(scope: OutlookResetScope) {
  if (!registerConfig.value) return
  const labels: Record<string, string> = {
    all: 'toate',
    failed: 'anormale',
    unused: 'nefolosite',
  }
  const ok = await confirmDialog.confirm({
    title: `Resetează emailuri ${labels[scope] || ''}`,
    message: `Sigur resetezi emailurile ${labels[scope] || ''} din pool?`,
    confirmText: 'Resetează',
    cancelText: 'Anulează',
    tone: 'danger',
  })
  if (!ok) return
  try {
    await registerApi.resetOutlookPool(scope)
    toast.success('Pool resetat')
    await loadLegacyConfig()
  } catch (error: any) {
    toast.error(error?.message || 'Eroare la resetare pool')
  }
}

function startPolling() {
  stopPolling()
  pollTimer.value = window.setInterval(() => {
    loadLegacyConfig()
  }, 5000)
}

function stopPolling() {
  if (pollTimer.value) {
    window.clearInterval(pollTimer.value)
    pollTimer.value = null
  }
}

function startSSE() {
  stopSSE()
  const token = getAuthToken()
  const url = `${registerApi.baseURL}/legacy/sse${token ? `?token=${encodeURIComponent(token)}` : ''}`
  eventSource.value = new EventSource(url)
  eventSource.value.onmessage = (e) => {
    try {
      const data = JSON.parse(e.data)
      if (data.type === 'log' && registerConfig.value) {
        registerConfig.value.logs = [...(registerConfig.value.logs || []), data]
        if (registerConfig.value.logs.length > 200) {
          registerConfig.value.logs = registerConfig.value.logs.slice(-200)
        }
      }
      if (data.type === 'stats' && registerConfig.value) {
        registerConfig.value.stats = { ...registerConfig.value.stats, ...data }
      }
    } catch {
      // ignore
    }
  }
  eventSource.value.onerror = () => {
    stopSSE()
    setTimeout(startSSE, 5000)
  }
}

function stopSSE() {
  if (eventSource.value) {
    eventSource.value.close()
    eventSource.value = null
  }
}

onMounted(() => {
  loadLegacyConfig()
  loadProxyGroups()
  startPolling()
  startSSE()
})

onBeforeUnmount(() => {
  stopPolling()
  stopSSE()
  clearAllGptMailRefreshTimers()
})
</script>

<style scoped>
.register-page {
  padding: 1rem;
}

.register-layout {
  display: grid;
  grid-template-columns: 1fr 380px;
  gap: 1.5rem;
}

@media (max-width: 1024px) {
  .register-layout {
    grid-template-columns: 1fr;
  }
}

.register-config-column {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.register-runtime-column {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.register-form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.75rem;
}

.register-form-grid--mail {
  grid-template-columns: 1fr 1fr 1fr;
}

.register-form-grid--two {
  grid-template-columns: 1fr 1fr;
}

.register-form-grid--three {
  grid-template-columns: 1fr 1fr 1fr;
}

.register-field {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.register-field--full {
  grid-column: 1 / -1;
}

.register-label {
  font-size: 0.75rem;
  font-weight: 500;
  color: var(--color-muted-foreground);
}

.register-checkbox-field {
  display: flex;
  align-items: center;
  padding-top: 0.5rem;
}

.register-checkbox-field--compact {
  padding-top: 0;
}

.register-proxy-hint {
  font-size: 0.75rem;
  color: var(--color-muted-foreground);
  margin-top: 0.25rem;
}

.register-provider-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.register-provider-card {
  border-radius: 0.75rem;
}

.register-provider-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
}

.register-provider-title {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-weight: 500;
  font-size: 0.875rem;
}

.register-provider-actions {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex-shrink: 0;
}

.register-provider-actions--left {
  justify-content: flex-start;
}

.register-provider-message {
  margin-top: 0.5rem;
}

.register-provider-section {
  margin-top: 0.75rem;
}

.register-provider-section--soft {
  background: var(--color-background);
  border-radius: 0.5rem;
  padding: 0.75rem;
  margin-left: -0.25rem;
  margin-right: -0.25rem;
}

.register-provider-section-title {
  font-size: 0.75rem;
  font-weight: 600;
  color: var(--color-foreground);
  margin-bottom: 0.5rem;
}

.register-provider-stack {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.register-gptmail-panel {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.register-gptmail-summary {
  display: flex;
  flex-wrap: wrap;
  gap: 0.375rem;
}

.register-textarea {
  width: 100%;
  min-height: 5rem;
  padding: 0.5rem;
  border-radius: 0.5rem;
  border: 1px solid var(--color-border);
  background: var(--color-background);
  font-family: var(--font-mono);
  font-size: 0.75rem;
  resize: vertical;
}

.register-textarea--tall {
  min-height: 8rem;
}

.register-preview-line {
  font-size: 0.75rem;
  color: var(--color-muted-foreground);
  margin-top: 0.25rem;
}

.register-outlook-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  margin-top: 0.5rem;
}

.register-outlook-summary {
  display: flex;
  flex-wrap: wrap;
  gap: 0.375rem;
}

.register-outlook-details {
  margin-top: 0.5rem;
}

.register-outlook-details summary {
  cursor: pointer;
  font-size: 0.75rem;
  font-weight: 500;
  color: var(--color-muted-foreground);
}

.register-outlook-detail-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 0.375rem;
  margin-top: 0.5rem;
}

.register-runtime-section {
  position: sticky;
  top: 1rem;
}

.register-runtime-actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.5rem;
}

.register-runtime-tips {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.register-runtime-tips p {
  font-size: 0.75rem;
  line-height: 1.4;
}

.register-runtime-log {
  margin-top: 0.5rem;
}
</style>