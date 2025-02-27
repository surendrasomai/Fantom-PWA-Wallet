<template>
    <div class="tx-confirmation">
        <f-card
            class="f-card-double-padding f-data-layout"
            :class="[windowMode ? 'column-layout column-layout--body-footer' : '']"
            :off="cardOff"
        >
            <div :class="{ 'center-v': windowMode }">
                <slot></slot>
            </div>

            <div>
                <ledger-message :error="error" @ledger-status-code="onLedgerStatusCode" />

                <transaction-confirmation-form
                    v-if="!hideTxForm"
                    :error-message="errorMsg"
                    :show-password-field="
                        !currentAccount.isLedgerAccount &&
                        !currentAccount.isMetamaskAccount &&
                        !currentAccount.isCoinbaseAccount &&
                        !currentAccount.isWalletConnectAccount
                    "
                    :password-label="passwordLabel"
                    :send-button-label="sendButtonLabel"
                    :waiting="waiting"
                    :disabled-submit="disabledSubmit"
                    :gas-limit="dGasLimit"
                    :tmp-pwd-code="tmpPwdCode"
                    :cancel-button-label="cancelButtonLabel"
                    :show-cancel-button="showCancelButton"
                    @f-form-submit="onFFormSubmit"
                    @cancel-button-click="$emit('cancel-button-click', $event)"
                />
            </div>
        </f-card>

        <f-window
            v-if="currentAccount.isLedgerAccount"
            ref="confirmationWindow"
            modal
            title="Transaction Confirmation"
            style="max-width: 800px;"
            animation-in="scale-center-enter-active"
            animation-out="scale-center-leave-active"
        >
            <!--                <icon data="@/assets/svg/nano-s-confirm-tx.svg" width="300" height="91" />-->
            <div class="align-center"><img src="img/nano-s-confirm-tx.png" alt="fantom nano device" /><br /><br /></div>

            <p class="align-center">Please confirm this transaction on your Ledger device:</p>

            <slot name="window-content"></slot>
        </f-window>

        <f-window
            v-if="currentAccount.isMetamaskAccount"
            ref="metamaskNoticeWindow"
            modal
            title="Notice"
            class="double-body-padding"
            style="max-width: 560px;"
            animation-in="scale-center-enter-active"
            animation-out="scale-center-leave-active"
        >
            <div class="align-center">
                <div v-if="!$metamask.isInstalled()">
                    Metamask is not installed.
                </div>
                <div v-else-if="!$metamask.isCorrectChainId()">
                    Please, select Opera chain in Metamask.
                </div>
                <div v-else-if="metamaskAccount.toLowerCase() !== currentAccount.address.toLowerCase()">
                    Please, select account <b>{{ currentAccount.address }}</b> in Metamask.
                </div>
            </div>
        </f-window>

        <f-window
            v-if="currentAccount.isWalletConnectAccount"
            ref="walletConnectNoticeWindow"
            modal
            title="Notice"
            class="double-body-padding"
            style="max-width: 560px;"
            animation-in="scale-center-enter-active"
            animation-out="scale-center-leave-active"
        >
            <div class="align-center">
                <div v-if="!$walletConnect.isCorrectChainId()">
                    Please, select Fantom Opera chain.
                </div>
                <div v-else-if="walletConnectAccount.toLowerCase() !== currentAccount.address.toLowerCase()">
                    Please, select account <b>{{ currentAccount.address }}</b>
                </div>
            </div>
        </f-window>

        <coinbase-wallet-notice-window v-if="currentAccount.isCoinbaseAccount" ref="coinbaseNoticeWindow" />
    </div>
</template>

<script>
import FCard from '../core/FCard/FCard.vue';
import FWindow from '../core/FWindow/FWindow.vue';
import LedgerMessage from '../LedgerMessage/LedgerMessage.vue';
import TransactionConfirmationForm from '../forms/TransactionConfirmationForm.vue';
import { mapGetters, mapState } from 'vuex';
import gql from 'graphql-tag';
import { U2FStatus } from '../../plugins/fantom-nano.js';
import { UPDATE_ACCOUNT_BALANCE } from '../../store/actions.type.js';
import appConfig from '../../../app.config.js';
import CoinbaseWalletNoticeWindow from '@/components/windows/CoinbaseWalletNoticeWindow/CoinbaseWalletNoticeWindow.vue';
import { cloneObject } from '@/utils';

/**
 * Base component for other 'transaction confirmation and send' components.
 */
export default {
    name: 'TxConfirmation',

    components: { CoinbaseWalletNoticeWindow, TransactionConfirmationForm, LedgerMessage, FWindow, FCard },

    props: {
        /** Transaction object to send */
        tx: {
            type: Object,
            default() {
                return null;
            },
            required: true,
        },
        /** Name of confirmation component in cebab case */
        confirmationCompName: {
            type: String,
            default: '',
        },
        /** Label for button in TransactionConfirmationForm component */
        sendButtonLabel: {
            type: String,
            default: '',
        },
        /** Label for button in TransactionConfirmationForm component */
        cancelButtonLabel: {
            type: String,
            default: 'Cancel',
        },
        showCancelButton: {
            type: Boolean,
            default: false,
        },
        /** Label for password field in TransactionConfirmationForm component */
        passwordLabel: {
            type: String,
            default: '',
        },
        /**
         * Function called when transaction was successful
         * @param {object} _data
         */
        onSendTransactionSuccess: {
            type: Function,
            default: null,
        },
        /** Don't render card */
        cardOff: {
            type: Boolean,
            default: false,
        },
        /** Set temporary password */
        setTmpPwd: {
            type: Boolean,
            default: false,
        },
        /** Component is placed in FWindow */
        windowMode: {
            type: Boolean,
            default: false,
        },
        /** Hide transaction form */
        hideTxForm: {
            type: Boolean,
            default: false,
        },
        /** */
        tmpPwdCode: {
            type: String,
            default: '',
        },
        /** Count of usage of temporary password */
        tmpPwdCount: {
            type: Number,
            default: 1,
        },
    },

    data() {
        return {
            errorMsg: '',
            error: null,
            waiting: false,
            disabledSubmit: true,
            dGasLimit: '',
        };
    },

    computed: {
        ...mapState('metamask', {
            metamaskAccount: 'account',
            metamaskChainId: 'chainId',
        }),

        ...mapState('walletConnect', {
            walletConnectAccount: 'account',
            walletConnectChainId: 'chainId',
        }),

        ...mapGetters(['currentAccount']),
    },

    watch: {
        metamaskAccount() {
            if (this.areMetamaskParamsOk()) {
                this.$refs.metamaskNoticeWindow.hide();
            } else {
                this.$refs.metamaskNoticeWindow.show();
            }
        },

        metamaskChainId() {
            if (this.areMetamaskParamsOk()) {
                this.$refs.metamaskNoticeWindow.hide();
            } else {
                this.$refs.metamaskNoticeWindow.show();
            }
        },

        tx() {
            this.init();
        },
    },

    methods: {
        async init() {
            this.dGasLimit = this.tx.gasLimit;
            this.disabledSubmit = false;
        },

        sendTransaction(_rawTransaction) {
            this.$apollo
                .mutate({
                    mutation: gql`
                        mutation($tx: Bytes!) {
                            sendTransaction(tx: $tx) {
                                hash
                                from
                                to
                            }
                        }
                    `,
                    variables: {
                        tx: _rawTransaction,
                    },
                })
                .then((_data) => {
                    if (this.onSendTransactionSuccess) {
                        this.onSendTransactionSuccess(_data);
                    }
                })
                .catch((_error) => {
                    this.errorMsg = _error;
                });
        },

        async onFFormSubmit(_event) {
            const { currentAccount } = this;
            const fWallet = this.$fWallet;
            const pwd = _event.detail.data.pwd;
            let rawTx = null;

            _event.detail.data.pwd = '';

            if (currentAccount && this.tx && fWallet.isValidAddress(this.tx.to)) {
                this.tx.nonce = await fWallet.getTransactionCount(currentAccount.address);
                this.tx.nonce = `0x${this.tx.nonce.toString(16)}`;
                this.tx.chainId = appConfig.chainId;

                if (!this.tx.gas) {
                    this.errorMsg = this.tx._error || 'Transaction Error';
                    return;
                }

                delete this.tx._error;
                delete this.tx._fee;
                // console.log('tx', this.tx);
                // console.log(currentAccount);

                if (currentAccount.keystore) {
                    delete this.tx.gasLimit;

                    if (pwd || fWallet.pwdStorage.isSet(this.tmpPwdCode)) {
                        try {
                            rawTx = await fWallet.signTransaction(
                                cloneObject(this.tx),
                                currentAccount.keystore,
                                pwd,
                                this.tmpPwdCode
                            );

                            if (this.setTmpPwd && this.tmpPwdCount > 0) {
                                fWallet.pwdStorage.set(pwd, this.tmpPwdCount, this.tmpPwdCode);
                            }
                        } catch (_error) {
                            console.error(_error);
                            this.errorMsg = _error.toString();
                            // this.errorMsg = 'Invalid password';
                        }
                    }
                } else if (currentAccount.isLedgerAccount) {
                    delete this.tx.gas;

                    try {
                        this.$refs.confirmationWindow.show();

                        rawTx = await this.$fNano.signTransaction(
                            cloneObject(this.tx),
                            currentAccount.accountId,
                            currentAccount.addressId
                        );

                        this.$refs.confirmationWindow.hide('fade-leave-active');
                    } catch (_error) {
                        this.error = _error;
                        this.$refs.confirmationWindow.hide();
                        // this.errorMsg = _error.toString();
                    }
                } else if (currentAccount.isMetamaskAccount) {
                    if (this.areMetamaskParamsOk()) {
                        const from = currentAccount.address;
                        const to = this.tx.to;

                        this.waiting = true;
                        const txHash = await this.$metamask.signTransaction(
                            cloneObject(this.tx),
                            currentAccount.address
                        );

                        if (this.onSendTransactionSuccess && txHash) {
                            this.onSendTransactionSuccess({
                                data: {
                                    sendTransaction: {
                                        hash: txHash,
                                        from,
                                        to,
                                    },
                                },
                            });
                        }
                    } else {
                        this.$refs.metamaskNoticeWindow.show();
                    }

                    this.waiting = false;
                } else if (currentAccount.isCoinbaseAccount) {
                    if (!this.areWalletlinkParamsOk()) {
                        this.$refs.coinbaseNoticeWindow.show();
                    } else {
                        try {
                            const from = currentAccount.address;
                            const to = this.tx.to;

                            this.waiting = true;
                            const txHash = await this.$walletlink.signTransaction(
                                cloneObject(this.tx),
                                currentAccount.address
                            );

                            if (this.onSendTransactionSuccess && txHash) {
                                this.onSendTransactionSuccess({
                                    data: {
                                        sendTransaction: {
                                            hash: txHash,
                                            from,
                                            to,
                                        },
                                    },
                                });
                            }
                        } catch (err) {
                            if (!this.$walletlink.selectedAddress) {
                                await this.$walletlink.connect();
                            }

                            this.waiting = false;
                        }
                    }
                } else if (currentAccount.isWalletConnectAccount) {
                    if (!this.areWalletConnectParamsOk()) {
                        await this.$walletConnect.connect();

                        if (!this.areWalletConnectParamsOk()) {
                            this.$refs.walletConnectNoticeWindow.show();
                        }
                    } else {
                        try {
                            const from = currentAccount.address;
                            const to = this.tx.to;

                            this.waiting = true;
                            const txHash = await this.$walletConnect.signTransaction(
                                cloneObject(this.tx),
                                currentAccount.address
                            );

                            if (this.onSendTransactionSuccess && txHash) {
                                this.onSendTransactionSuccess({
                                    data: {
                                        sendTransaction: {
                                            hash: txHash,
                                            from,
                                            to,
                                        },
                                    },
                                });
                            }
                        } catch (err) {
                            if (!this.$walletConnect.selectedAddress) {
                                await this.$walletConnect.connect();
                            }

                            this.waiting = false;
                        }
                    }
                }

                if (rawTx) {
                    // console.log('rawTx', rawTx);
                    this.sendTransaction(rawTx);

                    setTimeout(() => {
                        this.$store.dispatch(UPDATE_ACCOUNT_BALANCE);
                    }, 3000);
                }
            }
        },

        areMetamaskParamsOk() {
            return (
                this.$metamask.isInstalled() &&
                this.metamaskAccount.toLowerCase() === this.currentAccount.address.toLowerCase() &&
                this.$metamask.isCorrectChainId()
            );
        },

        areWalletlinkParamsOk() {
            const { $walletlink } = this;

            return (
                $walletlink.selectedAddress &&
                $walletlink.selectedAddress.toLowerCase() === this.currentAccount.address.toLowerCase() &&
                $walletlink.isCorrectChainId()
            );
        },

        areWalletConnectParamsOk() {
            const { $walletConnect } = this;

            return (
                $walletConnect.selectedAddress &&
                $walletConnect.selectedAddress.toLowerCase() === this.currentAccount.address.toLowerCase() &&
                $walletConnect.isCorrectChainId()
            );
        },

        /**
         * Triggered on 'ledger-status-code' event.
         *
         * @param {string} _code
         */
        onLedgerStatusCode(_code) {
            if (_code === U2FStatus.USER_REJECTED_REQUESTED_ACTION) {
                this.$emit('change-component', {
                    to: 'transaction-reject-message',
                    from: this.confirmationCompName,
                });
            }
        },
    },
};
</script>
