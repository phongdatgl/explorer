<script lang="ts" setup>
import {
    useBaseStore,
    useBlockchain,
    useFormatter,
    useMintStore,
    useStakingStore,
    useTxDialog,
} from '@/stores';
import { computed } from '@vue/reactivity';
import { onMounted, ref } from 'vue';
import { Icon } from '@iconify/vue';
import type { Key, SlashingParam, Validator } from '@/types';
import { formatSeconds}  from '@/libs/utils'

const staking = useStakingStore();
const base = useBaseStore();
const format = useFormatter();
const dialog = useTxDialog();
const chainStore = useBlockchain();
const mintStore = useMintStore()

const cache = JSON.parse(localStorage.getItem('avatars') || '{}');
const avatars = ref(cache || {});
const latest = ref({} as Record<string, number>);
const yesterday = ref({} as Record<string, number>);
const tab = ref('active');
const unbondList = ref([] as Validator[]);
const slashing =ref({} as SlashingParam)

interface myStakeInterface {
  "logo": string,
  "v": {
    operator_address: any,
    description: {
      moniker: any,
      website: any,
      identity: any,
      token: any,
      delegator_shares: any,
      consensus_pubkey: any,
    },
    commission: {
      commission_rates: {
        rate: any,
      }
    },
    jailed: any,
    tokens: any,
    delegator_shares: any,
    consensus_pubkey: any
  }
}
const myStake = ref({
  "logo": '',
  "v": {
    operator_address: '',
    description: {
      moniker: '',
      website: '',
      identity: '',
      token: '',
      delegator_shares: '',
      consensus_pubkey: '',
    },
    commission: {
      commission_rates: {
        rate: '',
      }
    },
    jailed: '',
    tokens: '',
    delegator_shares: '',
    consensus_pubkey: {
        '@type': '',
        key: ''
    }
  }
} as myStakeInterface);




onMounted(() => {
    staking.fetchUnbondingValdiators().then((res) => {
        unbondList.value = res.concat(unbondList.value);
    });
    staking.fetchInacitveValdiators().then((res) => {
        unbondList.value = unbondList.value.concat(res);
    });
    chainStore.rpc.getSlashingParams().then(res => {
        slashing.value = res.params
    })
});

async function fetchChange() {
    let page = 0;

    let height = Number(base.latest?.block?.header?.height || 0);
    if (height > 14400) {
        height -= 14400;
    } else {
        height = 1;
    }
    // voting power in 24h ago
    while (page < staking.validators.length && height > 0) {
        await base.fetchValidatorByHeight(height, page).then((x) => {
            x.validators.forEach((v) => {
                yesterday.value[v.pub_key.key] = Number(v.voting_power);
            });
        });
        page += 100;
    }

    page = 0;
    // voting power for now
    while (page < staking.validators.length) {
        await base.fetchLatestValidators(page).then((x) => {
            x.validators.forEach((v) => {
                latest.value[v.pub_key.key] = Number(v.voting_power);
            });
        });
        page += 100;
    }
}


const changes = computed(() => {
    const changes = {} as Record<string, number>;
    Object.keys(latest.value).forEach((k) => {
        const l = latest.value[k] || 0;
        const y = yesterday.value[k] || 0;
        changes[k] = l - y;
    });
    return changes;
});

const change24 = (key: Key) => {
    const txt = key.key;
    // const n: number = latest.value[txt];
    // const o: number = yesterday.value[txt];
    // // console.log( txt, n, o)
    // return n > 0 && o > 0 ? n - o : 0;
    return changes.value[txt];
};

const change24Text = (key?: Key) => {
    if (!key) return '';
    const v = change24(key);
    return v && v !== 0 ? format.showChanges(v) : '';
};

const change24Color = (key?: Key) => {
    if (!key) return '';
    const v = change24(key);
    if (v > 0) return 'text-success';
    if (v < 0) return 'text-error';
};

const calculateRank = function (position: number) {
    let sum = 0;
    for (let i = 0; i < position; i++) {
        sum += Number(staking.validators[i]?.delegator_shares);
    }
    const percent = sum / staking.totalPower;

    switch (true) {
        case tab.value === 'active' && percent < 0.33:
            return 'error';
        case tab.value === 'active' && percent < 0.67:
            return 'warning';
        default:
            return 'primary';
    }
};

function isFeatured(endpoints: string[], who?: {website?: string, moniker: string }) {
    if(!endpoints || !who) return false
    return endpoints.findIndex(x => who.website && who.website?.substring(0, who.website?.lastIndexOf('.')).endsWith(x) || who?.moniker?.toLowerCase().search(x.toLowerCase()) > -1) > -1
}

const list = computed(() => {
    if (tab.value === 'active') {
        return staking.validators.map((x, i) => ({v: x, rank: calculateRank(i), logo: logo(x.description.identity)}));
    } else if (tab.value === 'featured') {
        const endpoint = chainStore.current?.endpoints?.rest?.map(x => x.provider)
        if(endpoint) {
            endpoint.push('ping')
            return staking.validators
                .filter(x => isFeatured(endpoint, x.description))
                .map((x, i) => ({v: x, rank: 'primary', logo: logo(x.description.identity)}));
        }
        return []        
    }
    return unbondList.value.map((x, i) => ({v: x, rank: 'primary', logo: logo(x.description.identity)}));
});

const fetchAvatar = (identity: string) => {
  // fetch avatar from keybase
  return new Promise<void>((resolve) => {
    staking
      .keybase(identity)
      .then((d) => {
        if (Array.isArray(d.them) && d.them.length > 0) {
          const uri = String(d.them[0]?.pictures?.primary?.url).replace(
            'https://s3.amazonaws.com/keybase_processed_uploads/',
            ''
          );

          avatars.value[identity] = uri;
          resolve();
        } else throw new Error(`failed to fetch avatar for ${identity}`);
      })
      .catch((error) => {
        // console.error(error); // uncomment this if you want the user to see which avatars failed to load.
        resolve();
      });
  });
};

const loadAvatar = (identity: string) => {
  // fetches avatar from keybase and stores it in localStorage
  fetchAvatar(identity).then(() => {
    localStorage.setItem('avatars', JSON.stringify(avatars.value));
  });
};

const loadAvatars = () => {
  // fetches all avatars from keybase and stores it in localStorage
  const promises = staking.validators.map((validator) => {
    const identity = validator.description?.identity;

    // Here we also check whether we haven't already fetched the avatar
    if (identity && !avatars.value[identity]) {
      return fetchAvatar(identity);
    } else {
      return Promise.resolve();
    }
  });

  Promise.all(promises).then(() =>
    localStorage.setItem('avatars', JSON.stringify(avatars.value))
  );
};

const logo = (identity?: string) => {
    if (!identity || !avatars.value[identity]) return '';
    const url = avatars.value[identity] || '';
    return url.startsWith('http')
        ? url
        : `https://s3.amazonaws.com/keybase_processed_uploads/${url}`;
};

let height_in_24h = ref(0)
base.$subscribe((_, s) => {
    if (Number(s.earlest.block.header.height) !== height_in_24h.value) {
        height_in_24h.value = Number(s.earlest.block.header.height);
        fetchChange();
    }
});

loadAvatars();

const is_staked = computed(() => {
    let flag = false;
    staking.validators.forEach((v, index) => {
        if(v.description?.moniker==='d_validator') {
            myStake.value.logo = logo(v.description?.identity);
            myStake.value.v.operator_address = v.operator_address;
            myStake.value.v.description.moniker = v.description.moniker;
            myStake.value.v.description.website = v.description.website;
            myStake.value.v.description.identity = v.description.identity;
            myStake.value.v.tokens = v.tokens;
            myStake.value.v.consensus_pubkey.key = v.consensus_pubkey.key;
            myStake.value.v.consensus_pubkey['@type'] = v.consensus_pubkey['@type'];
            myStake.value.v.jailed = v.jailed;
            myStake.value.v.delegator_shares = v.delegator_shares;
            myStake.value.v.commission.commission_rates.rate = v.commission.commission_rates.rate;
            flag = true;
        }
    });

    unbondList.value.forEach((v, index) => {
        if(v.description?.moniker==='d_validator') {
            myStake.value.logo = logo(v.description?.identity);
            myStake.value.v.operator_address = v.operator_address;
            myStake.value.v.description.moniker = v.description.moniker;
            myStake.value.v.description.website = v.description.website;
            myStake.value.v.description.identity = v.description.identity;
            myStake.value.v.tokens = v.tokens;
            myStake.value.v.consensus_pubkey.key = v.consensus_pubkey.key;
            myStake.value.v.consensus_pubkey['@type'] = v.consensus_pubkey['@type'];
            myStake.value.v.jailed = v.jailed;
            myStake.value.v.delegator_shares = v.delegator_shares;
            myStake.value.v.commission.commission_rates.rate = v.commission.commission_rates.rate;
            flag = true;
        }
    });

    return flag;
});

</script>
<template>
<div>
    <div class="bg-base-100 rounded-lg grid sm:grid-cols-1 md:grid-cols-4 p-4" >    
        <div class="flex">
            <span>
                <div class="relative w-9 h-9 rounded overflow-hidden flex items-center justify-center mr-2">
                    <Icon class="text-success" icon="mdi:trending-up" size="32" />
                    <div class="absolute top-0 left-0 bottom-0 right-0 opacity-20 bg-success"></div>
                </div>
            </span>
            <span>
                <div class="font-bold">{{ format.percent(mintStore.inflation) }}</div>
                <div class="text-xs">{{ $t('staking.inflation') }}</div>
            </span>
        </div>
        <div class="flex">
            <span>
                <div class="relative w-9 h-9 rounded overflow-hidden flex items-center justify-center mr-2">
                    <Icon class="text-primary" icon="mdi:lock-open-outline" size="32" />
                    <div class="absolute top-0 left-0 bottom-0 right-0 opacity-20 bg-primary"></div>
                </div>
            </span>
            <span>
                <div class="font-bold">{{ formatSeconds(staking.params?.unbonding_time) }}</div>
                <div class="text-xs">{{ $t('staking.unbonding_time') }}</div>
            </span>
        </div> 
        <div class="flex">
            <span>
                <div class="relative w-9 h-9 rounded overflow-hidden flex items-center justify-center mr-2">
                    <Icon class="text-error" icon="mdi:alert-octagon-outline" size="32" />
                    <div class="absolute top-0 left-0 bottom-0 right-0 opacity-20 bg-error"></div>
                </div>
            </span>
            <span>
            <div class="font-bold">{{ format.percent(slashing.slash_fraction_double_sign) }}</div>
            <div class="text-xs">{{ $t('staking.double_sign_slashing') }}</div>
            </span>
        </div> 
        <div class="flex">
            <span>
                <div class="relative w-9 h-9 rounded overflow-hidden flex items-center justify-center mr-2">
                    <Icon class="text-error" icon="mdi:pause" size="32" />
                    <div class="absolute top-0 left-0 bottom-0 right-0 opacity-20 bg-error"></div>
                </div>
            </span>
            <span>
            <div class="font-bold">{{ format.percent(slashing.slash_fraction_downtime) }}</div>
            <div class="text-xs">{{ $t('staking.downtime_slashing') }}</div>
            </span>
        </div>  
    </div>

    <div class="bg-base-100 rounded-lg grid sm:grid-cols-1 p-4 mt-4" v-if="is_staked">
        <h3 class="font-bold">Stake with Us</h3>
        <div class="flex">
            <table class="table staking-table w-full">

                <tbody>
                    <tr class="hover:bg-gray-100 dark:hover:bg-[#384059]">
                            <!-- 👉 rank -->
                            <td style="width: 2.79883382%">
                                <div
                                    class="text-xs truncate relative px-2 py-1 rounded-full w-fit"
                                    :class="`text-error`"
                                >
                                    <span
                                        class="inset-x-0 inset-y-0 opacity-10 absolute"
                                        :class="`bg-error`"
                                    ></span>
                                    1
                                </div>
                            </td>
                            <!-- 👉 Validator -->
                            <td style="width: 41.6163265%">
                                <div
                                    class="flex items-center overflow-hidden"
                                    style="max-width: 300px"
                                >
                                    <div
                                        class="avatar mr-4 relative w-8 h-8 rounded-full"
                                    >
                                        <div
                                            class="w-8 h-8 rounded-full bg-gray-400 absolute opacity-10"
                                        ></div>
                                        <div class="w-8 h-8 rounded-full">
                                            <img
                                                v-if="myStake.logo"
                                                :src="myStake.logo"
                                                class="object-contain"
                                                
                                            />
                                            <Icon
                                                v-else
                                                class="text-3xl"
                                                :icon="`mdi-help-circle-outline`"
                                            />
                                            
                                        </div>
                                    </div>

                                    <div class="flex flex-col">
                                        <span class="text-sm text-primary dark:invert whitespace-nowrap overflow-hidden">
                                            <RouterLink
                                                :to="{
                                                    name: 'chain-staking-validator',
                                                    params: {
                                                        validator:
                                                            myStake.v.operator_address,
                                                    },
                                                }"
                                                class="font-weight-medium"
                                            >
                                                {{ myStake.v.description?.moniker }}
                                            </RouterLink>
                                        </span>
                                        <span class="text-xs">{{
                                            myStake.v.description?.website ||
                                            myStake.v.description?.identity ||
                                            '-'
                                        }}</span>
                                    </div>
                                </div>
                            </td>

                            <!-- 👉 Voting Power -->
                            <td class="text-right" style="width: 19.5451895%">
                                <div class="flex flex-col">
                                    <h6 class="text-sm font-weight-medium whitespace-nowrap ">
                                        {{
                                            format.formatToken(
                                                {
                                                    amount: parseInt(
                                                        myStake.v.tokens
                                                    ).toString(),
                                                    denom: staking.params
                                                        .bond_denom,
                                                },
                                                true,
                                                '0,0'
                                            )
                                        }}
                                    </h6>
                                    <span class="text-xs">{{
                                        format.calculatePercent(
                                            myStake.v.delegator_shares,
                                            staking.totalPower
                                        )
                                    }}</span>
                                </div>
                            </td>
                            <!-- 👉 24h Changes -->
                            <td
                                class="text-right text-xs"
                                :class="change24Color(myStake.v.consensus_pubkey)"
                                style="width: 13.0693878%"
                            >
                                {{ change24Text(myStake.v.consensus_pubkey) }}&nbsp;
                            </td>
                            <!-- 👉 commission -->
                            <td class="text-right text-xs" style="width: 12.2204082%">
                                {{
                                    format.formatCommissionRate(
                                        myStake.v.commission?.commission_rates?.rate
                                    )
                                }}
                            </td>
                            <!-- 👉 Action -->
                            <td class="text-center">
                                <div
                                    v-if="myStake.v.jailed"
                                    class="badge badge-error gap-2 text-white"
                                >
                                {{ $t('staking.jailed') }}
                                </div>
                                <label
                                    v-else
                                    for="delegate"
                                    class="btn btn-xs btn-primary rounded-sm capitalize"
                                    @click="
                                        dialog.open('delegate', {
                                            validator_address:
                                                myStake.v.operator_address,
                                        })
                                    "
                                    >{{ $t('account.btn_delegate') }}</label
                                >
                            </td>

                    </tr>
                </tbody>
            </table>
        </div>
        
    </div>

    <div>
        <div class="flex items-center justify-between py-1">
            <div class="tabs tabs-boxed bg-transparent">
                <a
                    class="tab text-gray-400"
                    :class="{ 'tab-active': tab === 'featured' }"
                    @click="tab = 'featured'"
                    >{{ $t('staking.popular') }}</a
                >
                <a
                    class="tab text-gray-400"
                    :class="{ 'tab-active': tab === 'active' }"
                    @click="tab = 'active'"
                    >{{ $t('staking.active') }}</a
                >
                <a
                    class="tab text-gray-400"
                    :class="{ 'tab-active': tab === 'inactive' }"
                    @click="tab = 'inactive'"
                    >{{ $t('staking.inactive') }}</a
                >
            </div>

            <div class="text-lg font-semibold">
                {{ list.length }}/{{ staking.params.max_validators }}
            </div>
        </div>

        <div class="bg-base-100 px-4 pt-3 pb-4 rounded shadow">
            <div class="overflow-x-auto">
                <table class="table staking-table w-full">
                    <thead>
                        <tr>
                            <th
                                scope="col"
                                class="uppercase"
                                style="width: 3rem; position: relative"
                            >
                            {{ $t('staking.rank') }}    
                            </th>
                            <th scope="col" class="uppercase">{{ $t('staking.validator') }}</th>
                            <th scope="col" class="text-right uppercase">{{ $t('staking.voting_power') }}</th>
                            <th scope="col" class="text-right uppercase">{{ $t('staking.24h_changes') }}</th>
                            <th scope="col" class="text-right uppercase">{{ $t('staking.commission') }}</th>
                            <th scope="col" class="text-center uppercase">{{ $t('staking.actions') }}</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr
                            v-for="({v, rank, logo}, i) in list"
                            :key="v.operator_address"
                            class="hover:bg-gray-100 dark:hover:bg-[#384059]"
                        >
                            <!-- 👉 rank -->
                            <td>
                                <div
                                    class="text-xs truncate relative px-2 py-1 rounded-full w-fit"
                                    :class="`text-${rank}`"
                                >
                                    <span
                                        class="inset-x-0 inset-y-0 opacity-10 absolute"
                                        :class="`bg-${rank}`"
                                    ></span>
                                    {{ i + 1 }}
                                </div>
                            </td>
                            <!-- 👉 Validator -->
                            <td>
                                <div
                                    class="flex items-center overflow-hidden"
                                    style="max-width: 300px"
                                >
                                    <div
                                        class="avatar mr-4 relative w-8 h-8 rounded-full"
                                    >
                                        <div
                                            class="w-8 h-8 rounded-full bg-gray-400 absolute opacity-10"
                                        ></div>
                                        <div class="w-8 h-8 rounded-full">
                                            <img
                                                v-if="logo"
                                                :src="logo"
                                                class="object-contain"
                                                @error="
                                                    (e) => {
                                                        const identity = v.description?.identity;
                                                        if (identity) loadAvatar(identity);
                                                    }
                                                "
                                            />
                                            <Icon
                                                v-else
                                                class="text-3xl"
                                                :icon="`mdi-help-circle-outline`"
                                            />
                                            
                                        </div>
                                    </div>

                                    <div class="flex flex-col">
                                        <span class="text-sm text-primary dark:invert whitespace-nowrap overflow-hidden">
                                            <RouterLink
                                                :to="{
                                                    name: 'chain-staking-validator',
                                                    params: {
                                                        validator:
                                                            v.operator_address,
                                                    },
                                                }"
                                                class="font-weight-medium"
                                            >
                                                {{ v.description?.moniker }}
                                            </RouterLink>
                                        </span>
                                        <span class="text-xs">{{
                                            v.description?.website ||
                                            v.description?.identity ||
                                            '-'
                                        }}</span>
                                    </div>
                                </div>
                            </td>

                            <!-- 👉 Voting Power -->
                            <td class="text-right">
                                <div class="flex flex-col">
                                    <h6 class="text-sm font-weight-medium whitespace-nowrap ">
                                        {{
                                            format.formatToken(
                                                {
                                                    amount: parseInt(
                                                        v.tokens
                                                    ).toString(),
                                                    denom: staking.params
                                                        .bond_denom,
                                                },
                                                true,
                                                '0,0'
                                            )
                                        }}
                                    </h6>
                                    <span class="text-xs">{{
                                        format.calculatePercent(
                                            v.delegator_shares,
                                            staking.totalPower
                                        )
                                    }}</span>
                                </div>
                            </td>
                            <!-- 👉 24h Changes -->
                            <td
                                class="text-right text-xs"
                                :class="change24Color(v.consensus_pubkey)"
                            >
                                {{ change24Text(v.consensus_pubkey) }}
                            </td>
                            <!-- 👉 commission -->
                            <td class="text-right text-xs">
                                {{
                                    format.formatCommissionRate(
                                        v.commission?.commission_rates?.rate
                                    )
                                }}
                            </td>
                            <!-- 👉 Action -->
                            <td class="text-center">
                                <div
                                    v-if="v.jailed"
                                    class="badge badge-error gap-2 text-white"
                                >
                                {{ $t('staking.jailed') }}
                                </div>
                                <label
                                    v-else
                                    for="delegate"
                                    class="btn btn-xs btn-primary rounded-sm capitalize"
                                    @click="
                                        dialog.open('delegate', {
                                            validator_address:
                                                v.operator_address,
                                        })
                                    "
                                    >{{ $t('account.btn_delegate') }}</label
                                >
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>

            <div class="divider"></div>
            <div class="flex flex-row items-center">
                <div
                    class="text-xs truncate relative py-2 px-4 rounded-md w-fit text-error mr-2"
                >
                    <span
                        class="inset-x-0 inset-y-0 opacity-10 absolute bg-error"
                    ></span>
                    {{ $t('staking.top') }} 33%
                </div>
                <div
                    class="text-xs truncate relative py-2 px-4 rounded-md w-fit text-warning"
                >
                    <span
                        class="inset-x-0 inset-y-0 opacity-10 absolute bg-warning"
                    ></span>
                    {{ $t('staking.top') }} 67%
                </div>
                <div class="text-xs hidden md:!block pl-2">
                    {{ $t('staking.description') }}
                </div>
            </div>
        </div>
    </div>
</div>
</template>

<route>
  {
    meta: {
      i18n: 'staking',
      order: 3
    }
  }
</route>

<style>
.staking-table.table :where(th, td) {
    padding: 8px 5px;
    background: transparent;
}
</style>
