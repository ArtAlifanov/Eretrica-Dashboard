<template>
    <div class="action-group">
        
        <!-- <el-button @click="saveAndClose" size="mini" type="primary" class="el-button--transparent"> {{ this.$t('general.actions.save_and_close') }}
        </el-button> -->
        <el-button
            v-if="(deleteAction || undefined) && editMode"
            @click="deleteAndClose"
            size="mini"
            type="danger"
            class="el-button--transparent"
        >
            <!-- <svg-icon icon="trash" /> 
            &nbsp; -->
            {{this.$t('general.actions.delete')}}
        </el-button>
        <el-button
            v-if="editMode"
            @click="$emit('edit-mode', true)"
            size="mini"
        >
            <!-- <svg-icon icon="save" /> 
            &nbsp; -->
            {{this.$t('general.actions.cancel')}}
        </el-button>
        <el-button
            v-if="(editMode || deleteAction === undefined) && showSaveBtn"
            @click="SaveAndEdit"
            size="mini"
            type="success"
        >
            <!-- <svg-icon icon="save" /> 
            &nbsp; -->
            {{this.$t('general.actions.save')}}
        </el-button>
        <!-- <el-button @click="goToListing" size="mini" type="warning" class="el-button--transparent"> {{this.$t('general.actions.close')}}
        </el-button> -->
        <el-button
            v-if="editMode"
            @click="$emit('edit-mode')"
            size="mini"
            icon="el-icon-close"
            type="info"
            circle
            class="el-button--transparent"
        ></el-button>
        <el-button
            v-if="!editMode && deleteAction !== undefined"
            @click="$emit('edit-mode') "
            size="mini"
            type="danger"
            class="el-button--transparent"
        >
            <!-- <svg-icon icon="edit" /> 
            &nbsp; -->
            {{this.$t('general.actions.update')}}
        </el-button>
        <el-button
            v-if="!editMode"
            @click="goToListing"
            size="mini"
            icon="el-icon-close"
            type="info"
            circle
            class="el-button--transparent"
        ></el-button>
    </div>
</template>

<script>
    import {displayError, displaySuccess} from "helpers/messages";

    export default {
        props: {
            saveAction: {
                type: Function,
                required: true
            },
            deleteAction: {
                type: Function,
                required: false
            },
            route: {
                type: String,
                required: true
            },
            editRoute: {
                type: String,
                required: false
            },
            queryParams: { 
                type: Object,
                default() {
                    return {}
                }
            },
            role: {
                type: String
            },
            editMode: {
                type: Boolean,
                default() {
                    return false;
                }
            },
            showSaveBtn: {
                type: Boolean,
                default: true,
            }
        },
        methods: {
            goToListing() {
                let route = {};
                if(this.role) {
                    route = {
                        name: this.route,
                        query: {
                            role: this.role,
                            page: 1,
                            per_page: 20
                        }
                    }
                }
                else {
                    route = {
                        name: this.route,
                        query: this.queryParams
                    }
                }
                return this.$router.push(route);
            },
            async saveAndClose() {
                try {
                    const resp = await this.saveAction();
                    
                    if(resp) {
                        this.goToListing();
                    }
                } catch (e) {
                    console.log(e)
                }
            },
            deleteAndClose() {
                this.$confirm(this.$t('general.swal.delete.text'), this.$t('general.swal.delete.title'), {
                        type: 'warning'
                    }).then(() => {
                        this.callDeleteAction();
                    }).catch(() => {
                    });
            },
            async SaveAndEdit() {
                try {
                    const resp = await this.saveAction();
                    if (resp === false) {
                        return;
                    }
                    this.$emit('update:editMode', false);
                    if (resp && resp.data) {
                        this.$router.push({
                            name: this.editRoute,
                            params: {id: resp.data.id}
                        })
                    }
                } catch (e) {
                    console.log(e)
                }

            },
            async callDeleteAction() {
                const resp = await this.deleteAction({id: parseInt(this.$route.params.id)})
                    .then(r => {
                        displaySuccess(r);
                        this.goToListing();
                    })
                    .catch(err => displayError(err)); 
            }
        }
    }
</script>

<style lang="scss" scoped>
    .action-group > .el-button:not(:first-child) {
        margin-left: 0px;
    }
    /deep/ .el-icon-close {
        color: #3d3f41;
    }
</style>
