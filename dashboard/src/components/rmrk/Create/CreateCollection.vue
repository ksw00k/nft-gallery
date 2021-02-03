<template>
  <div>
    <b-loading is-full-page v-model="isLoading" :can-cancel="true"></b-loading>
    <div>Using {{ version }}</div>
    <div>
      Computed id: <b>{{ rmrkId }}</b>
    </div>
    <AccountSelect label="Account" v-model="accountId" />
    <b-field label="Name">
      <b-input v-model="rmrkMint.name"></b-input>
    </b-field>
    <b-field label="Maximum NFTs in collection">
      <b-numberinput
        v-model="rmrkMint.max"
        placeholder="1 is minumum"
        :min="1"
      ></b-numberinput>
    </b-field>
    <b-field label="Symbol">
      <b-input v-model="rmrkMint.symbol"></b-input>
    </b-field>
    <b-switch v-model="uploadMode" passive-type="is-dark" type="is-info">
      {{ uploadMode ? 'Upload' : 'IPFS hash' }}
    </b-switch>
    <template v-if="uploadMode">
      <b-field label="Description">
        <b-input
          v-model="meta.description"
          maxlength="200"
          type="textarea"
        ></b-input>
      </b-field>
      <MetadataUpload v-model="image" />
      <b-field label="Image data">
        <b-input v-model="meta.image_data"></b-input>
      </b-field>
    </template>

    <b-field v-else label="Metadata IPFS Hash">
      <b-input v-model="rmrkMint.metadata"></b-input>
    </b-field>
    <PasswordInput v-model="password" :account="accountId" />   
    <b-button
      type="is-primary"
      icon-left="paper-plane"
      @click="submit"
      :disabled="disabled"
      :loading="isLoading"
    >
      Submit
    </b-button>
  </div>
</template>

<script lang="ts" >
import { Component, Prop, Vue, Watch } from 'vue-property-decorator';
import { RmrkMint } from '../types';
import { emptyObject } from '@/utils/empty';
import AccountSelect from '@/components/shared/AccountSelect.vue';
import MetadataUpload from './MetadataUpload.vue';
import Connector from '@vue-polkadot/vue-api';
import exec from '@/utils/transactionExecutor';
import { notificationTypes, showNotification } from '@/utils/notification';
import PasswordInput from '@/components/shared/PasswordInput.vue';

import { getInstance, RmrkType } from '../service/RmrkService';
import { Collection, CollectionMetadata } from '../service/scheme';
import { pinFile, pinJson, unSanitizeIpfsUrl } from '@/pinata';
import { decodeAddress } from '@polkadot/keyring';
import { u8aToHex } from '@polkadot/util';

const components = {
  AccountSelect,
  MetadataUpload,
  PasswordInput,
};

@Component({ components })
export default class CreateCollection extends Vue {
  private version: string = 'RMRK1.0.0';
  private rmrkMint: Collection = emptyObject<Collection>();
  private meta: CollectionMetadata = emptyObject<CollectionMetadata>();
  private accountId: string = '';
  private uploadMode: boolean = true;
  private image: Blob | null = null;
  private isLoading: boolean = false;
  private password: string = '';

  get rmrkId(): string {
    return this.generateId(this.accountIdToPubKey);
  }

  get accountIdToPubKey() {
    return (this.accountId && u8aToHex(decodeAddress(this.accountId))) || '';
  }

  private generateId(pubkey: string): string {
    return (
      pubkey?.substr(2, 10) +
      pubkey?.substring(pubkey.length - 8) +
      '-' +
      (this.rmrkMint?.symbol || '')
    ).toUpperCase();
  }

  get disabled(): boolean {
    const { name, symbol, max } = this.rmrkMint;
    return !(name && symbol && max && this.accountId && this.image);
  }

  public constructRmrkMint(): Collection {
    const mint: Collection = {
      ...this.rmrkMint,
      symbol: this.rmrkMint.symbol.toUpperCase(),
      version: this.version,
      issuer: this.accountId,
      metadata: unSanitizeIpfsUrl(this.rmrkMint?.metadata),
      id: this.rmrkId
    };

    return mint;
  }

  public async constructMeta() {
    if (!this.image) {
      throw new ReferenceError('No file found!');
    }

    this.meta = {
      ...this.meta,
      attributes: [],
      external_url: `https://rmrk.app/registry/${this.rmrkId}`
    };

    // TODO: upload image to IPFS
    const imageHash = await pinFile(this.image);
    this.meta.image = unSanitizeIpfsUrl(imageHash);
    // TODO: upload meta to IPFS
    const metaHash = await pinJson(this.meta);

    return unSanitizeIpfsUrl(metaHash);
  }

  private async submit() {
    this.isLoading = true;
    const { api } = Connector.getInstance();
    const rmrkService = getInstance();
    const mint = this.constructRmrkMint();
    if (!this.rmrkMint.metadata) {
      const meta = await this.constructMeta();
      mint.metadata = meta;
    }

    const mintString = `RMRK::MINT::${this.version}::${encodeURIComponent(
      JSON.stringify(mint)
    )}`;
    try {
      showNotification(mintString)
      console.log('submit', mintString);
      const tx = await exec(this.accountId, this.password, api.tx.system.remark, [
        mintString
      ]);
      console.warn('TX IN', tx);
      const persisted = await rmrkService?.resolve(mintString, this.accountId);
      console.log('SAVED', persisted?._id);
      showNotification(`[TEXTILE] ${persisted?._id}`, notificationTypes.success)
    } catch (e) {
      showNotification(`[ERR] ${e}`, notificationTypes.danger)
      console.error(e);
    }

    this.isLoading = false;
  }

  private upload(data: File) {
    console.log('upload', data.name);
    this.image = data;
  }
}
</script>

// {
//   "version": "RMRK0.1",
//   "name": "Genesis limited edition obxium art NFTs on Kusama blockchain",
//   "max": 5,
//   "issuer": "DmUVjSi8id22vcH26btyVsVq39p8EVPiepdBEYhzoLL8Qby",
//   "symbol": "OKSM",
//   "id": "241B8516516F381A-OKSM",
//   "metadata": "ipfs://ipfs/QmTcaAPWPY5NinmCdDJAi6YFmyag41UEy4SpE1jn4Xdhnx"
// }