<script>
import filter from "lodash/filter";
import empty from "lodash/isEmpty";
import forEach from "lodash/forEach";
import TopBarContent from "./subcomponents/TopBarContent.vue";
import runAjax from "../mixins/runAjax.js";

export default {
    name: "TopBarPanel",
    props: {
        initialsid: { type: Number | String, default: 0 },
        initialtype: { type: String, default: "" }
    },
    // TODO: Für jede Topbar muss eine eigene Struktur für die TopBarExtended erstellt werden.
    data: () => {
        return {
            hasRunOnce: false,
            counter: 0,
            slide: false,
            loading: true,
            slotbutton: null,
        };
    },
    computed: {
        qid: {
            get() {
                return this.$store.state.qid;
            },
            set(newValue) {
                this.$store.commit("setQid", newValue);
            }
        },
        gid: {
            get() {
                return this.$store.state.gid;
            },
            set(newValue) {
                this.$store.commit("setGid", newValue);
            }
        },
        sid: {
            get() {
                return this.$store.state.sid;
            },
            set(newValue) {
                this.$store.commit("setSid", newValue);
            }
        },
        type: {
            get() {
                return this.$store.state.type;
            },
            set(newValue) {
                this.$store.commit("setType", newValue);
            }
        },
        showSaveButton: {
            get() {
                return !!this.$store.state.showSaveButton;
            },
            set(newValue) {
                this.$store.commit("setShowSaveButton", newValue);
            }
        },
        showCloseButton: {
            get() {
                return !!this.$store.state.showCloseButton;
            },
            set(newValue) {
                this.$store.commit("setShowCloseButton", newValue);
            }
        },
        closeButtonUrl: {
            get() {
                return this.$store.state.closeButtonUrl;
            },
            set(newValue) {
                this.$store.commit("setCloseButtonUrl", newValue);
            }
        },
        topbarKey() {
            return "topbar-" + this.type + '-' + this.counter;
        },
        topbarExtendedKey() {
            return "topbar-extended-" + this.type + '-' + this.counter;
        },
        ownPermissions() {
            return this.$store.state.permissions;
        },
        getLeftButtons() {
            if (this.$store.state.topbar_left_buttons != null) {
                return filter(
                    this.$store.state.topbar_left_buttons,
                    button =>
                        !empty(button.name) || !empty(button.main_button.name)
                );
            }
            return [];
        },
        getRightButtons() {
            if (this.$store.state.topbar_right_buttons != null) {
                return filter(
                    this.$store.state.topbar_right_buttons,
                    button => (
                        (!button.isCloseButton && !button.isSaveButton)
                        || (this.showSaveButton && button.isSaveButton) 
                        || (this.showCloseButton && button.isCloseButton)
                    )
                );
            }
            return [];
        },
        getLeftButtonsExtended() {
            if (this.$store.state.topbarextended_left_buttons != null) {
                return filter(
                    this.$store.state.topbarextended_left_buttons,
                    button =>
                        !empty(button.name) || !empty(button.main_button.name)
                );
            }
            return [];
        },
        getRightButtonsExtended() {
            if (this.$store.state.topbarextended_right_buttons != null) {
                return filter(
                    this.$store.state.topbarextended_right_buttons,
                    button => (
                        (!button.isCloseButton && !button.isSaveButton)
                        || (this.showSaveButton && button.isSaveButton) 
                        || (this.showCloseButton && button.isCloseButton)
                    )
                );
            }
            return [];
        }
    },
    methods: {
        setType() {
            this.$log.log("Loading topbar contents based on type ", this.type);
            let dispatchAction = '';
            let errorHeader = "";
            switch(this.type) {
            case "question":
                dispatchAction = "getTopBarButtonsQuestion";
                errorHeader = "ERROR QUESTION";
                break;
            case "group":
                dispatchAction = "getTopBarButtonsGroup";
                errorHeader = "ERROR GROUP";
                break;
            case "survey":
                dispatchAction = "getTopBarButtonsSurvey";
                errorHeader = "ERROR SURVEY";
                this.slide = false;
                break;
            case "tokens":
                dispatchAction = "getTopBarButtonsTokens";
                errorHeader = "ERROR TOKEN";
                this.slide = false;
                break;
            case "responses":
                dispatchAction = "getTopBarButtonsResponses";
                errorHeader = "ERROR RESPONSES";
                this.slide = false;
                break;
            case "conditions":
                dispatchAction = "getTopBarButtonsConditions";
                errorHeader = "ERROR CONDITIONS";
                this.slide = false;
                break;
            }
            
            this.loading = true;
            this.$store.dispatch(dispatchAction)
                .then((data) => {
                    return this.$store.dispatch('getCustomTopbarContent').then((data2)=> { return [data, data2] });
                }).then(datas => {
                    this.$log.log("Promise resolved with datas", datas);
                    this.counter++;
                //LS.pageLoadActions.saveBindings();
                }).catch(error => {
                    this.$log.error(errorHeader);
                    this.$log.error(error);
                }).finally(() => {
                    this.$log.log("Trigger loading end");
                    this.loading = false;
                    this.$forceUpdate();
                    LS.pageLoadActions.saveBindings();
                });
        },

        onFade(slideable) {
            this.slide = slideable;
            LS.pageLoadActions.saveBindings();
        },
        unsetThis() {
            this.$log.log("unsetting");
            this.$store.commit("clean");
        },
        readGlobalObject(globalObject) {
            this.$log.log("Loading from global ->", globalObject);
            forEach(globalObject, (element,key) => {
                this[key] = element;    
            });
            this.closeButtonUrl =
                globalObject.closeButtonUrl ||
                LS.createUrl("admin/survey/sa/view/", { sid: this.sid });
        }
    },
    created() {
        
        this.sid = this.initialsid;
        this.type = this.initialtype;
        
        LS.EventBus.$on("doFadeEvent", slideable => {
            if (slideable == null) {
                slideable == !this.slide;
            }
            this.onFade(slideable);
        });

        LS.EventBus.$on("slotbuttonSet", payload => {
            this.slotbutton = payload.html || null;
        });

        LS.EventBus.$on("reloadTopBar", data => {
            this.$log.log("reloadTopBar triggered with -> ", data);
            const currentType = this.type;
            this.readGlobalObject(data);

            if (!this.hasRunOnce) {
                this.hasRunOnce = true;
                this.setType();
                return;
            }

            if (currentType === this.type) {
                this.loading = false;
            } 
            
            this.setType();
        });
    },
    render(h) {
        return (
            <div class="topbarpanel">
                <nav class="navbar navbar-default scoped-topbar-nav">
                    <transition name="slide-down">
                        {this.loading ? (
                            <loader-widget />
                        ) : this.slide ? ( //Extended first
                            <TopBarContent
                                key = {this.topbarKey}
                                leftButtons = {this.getLeftButtonsExtended}
                                rightButtons = {this.getRightButtonsExtended}
                            />
                        ) : ( //Regular topbar
                            <TopBarContent
                                key = {this.topbarExtendedKey}
                                leftButtons = {this.getLeftButtons}
                                rightButtons = {this.getRightButtons}
                                slotbutton = {this.slotbutton}
                            />
                        )}
                    </transition>
                </nav>
            </div>
        );
    }
};
</script>

<style lang="scss" scoped>
.topbarpanel {
    position: relative;
    padding-right: 6px;
    min-height: 50px;
}

.scoped-topbar-nav {
    .navbar {
        flex-wrap: wrap;
    }
    .navbar-btn {
        margin-top: 0;
    }
}

.navbar,
.navbar-default {
    padding-left: 1px;
    @media(max-width:1280px) {
        padding-left: 16px;
        padding-right: 16px;
    }
    
    border: none;
}

.scoped-switch-floats {
    .navbar-nav {
        li {
            float: right;
        }
    }
}

.nav > li {
    margin-left: 2px;
}

.padding-left {
    padding-left: 5px;
}

.right {
    align-self: flex-end;
}

</style>
