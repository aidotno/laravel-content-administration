<template>
    <div>
        <b-card
            no-body
            class="fj-index-table"
            :style="noCard ? 'box-shadow: none;' : ''"
        >
            <b-tabs card v-if="hasTabs" @activate-tab="newTab">
                <b-tab
                    :title="'title' in t ? t.title : t"
                    :active="t == tab"
                    v-for="(t, index) in tabs"
                    :key="index"
                    no-body
                />
            </b-tabs>
            <div class="card-body">
                <slot name="header" v-bind:tab="tab" />

                <fj-base-index-table-form
                    v-if="
                        searchKeys.length != 0 ||
                            !_.isEmpty(sortBy) ||
                            !_.isEmpty(filter)
                    "
                >
                    <b-input-group>
                        <b-input-group-prepend is-text>
                            <fa-icon icon="search" />
                        </b-input-group-prepend>

                        <fj-base-index-table-search
                            :nameSingular="nameSingular"
                            :namePlural="namePlural"
                            @search="doSearch"
                        />
                    </b-input-group>
                    <fj-base-index-table-filter
                        :filter="filter"
                        v-if="hasFilter"
                        @onFilterChange="filterChanged"
                    />

                    <fj-base-index-table-sort
                        :sortBy="sortBy"
                        v-if="hasSort"
                        :sortByDefault="sortByDefault"
                        @sort="sort"
                    />
                </fj-base-index-table-form>

                <fj-base-index-table-selected-items-actions
                    :actions="actions"
                    :items="items"
                    :selectedItems="selectedItems"
                    @reload="executedActions"
                    v-if="!!actions.length"
                />

                <fj-base-index-table
                    ref="table"
                    :namePlural="namePlural"
                    :sortable="canSort"
                    :busy="isBusy"
                    :cols="cols"
                    :items="items"
                    :radio="radio"
                    :no-head="noHead"
                    :no-select="noSelect"
                    :small="small"
                    :selectedItems="selectedItems"
                    @select="select"
                    @unselect="unselect"
                    @sort="sort"
                    @loadItems="_loadItems()"
                    @_sorted="sorted"
                    :class="{ paginated: total > items.length }"
                    v-on="$listeners"
                />

                <fj-base-index-table-index-indicator
                    :per-page="perPage"
                    :total="total"
                    :items="items"
                    :current-page="currentPage"
                    v-if="total > items.length"
                />

                <div
                    class="d-flex justify-content-center fj-index-table-pagination"
                    v-if="numberOfPages > 1"
                >
                    <b-pagination-nav
                        class="mt-2"
                        :link-gen="linkGen"
                        v-model="currentPage"
                        :number-of-pages="numberOfPages"
                        @change="goToPage"
                    />
                </div>
            </div>
        </b-card>
    </div>
</template>

<script>
export default {
    name: 'IndexTable',
    props: {
        sortable: {
            type: [Boolean, String],
            default() {
                return false;
            }
        },
        orderColumn: {
            type: String,
            default() {
                return 'order_column';
            }
        },
        tabs: {
            required: false,
            type: Array,
            default() {
                return [];
            }
        },
        radio: {
            type: Boolean,
            default() {
                return false;
            }
        },
        small: {
            type: Boolean,
            default() {
                return false;
            }
        },
        noSelect: {
            type: Boolean,
            default() {
                return false;
            }
        },
        noHead: {
            type: Boolean,
            default() {
                return false;
            }
        },
        noCard: {
            type: Boolean,
            default() {
                return false;
            }
        },
        cols: {
            required: true,
            type: Array
        },
        searchKeys: {
            type: Array,
            default() {
                return [];
            }
        },
        actions: {
            type: Array,
            default: () => {
                return [];
            }
        },
        items: {
            type: [Object, Array],
            required: true
        },
        loadItems: {
            type: Function,
            default() {
                null;
            }
        },
        nameSingular: {
            type: String
        },
        namePlural: {
            type: String
        },
        perPage: {
            type: Number,
            default() {
                return 20;
            }
        },
        filter: {
            type: [Object, Array],
            default() {
                return {};
            }
        },
        sortBy: {
            type: Object,
            default() {
                return {};
            }
        },
        sortByDefault: {
            type: String,
            default() {
                return 'id.desc';
            }
        },
        selected: {
            type: Array,
            default() {
                return [];
            }
        }
    },
    data() {
        return {
            isBusy: false,
            tab: null,
            search: '',
            sort_by_key: '',
            filter_scope: null,
            currentPage: 1,
            numberOfPages: 1,
            selectedItems: [],
            total: 0
        };
    },
    watch: {
        tabs() {
            if (!this.hasTabs) {
                return;
            }
            if (!this.tabs.includes(this.tab)) {
                this.tab = this.tabs[0];
            }
        }
    },
    beforeMount() {
        if (this.hasTabs) {
            this.tab = this.tabs[0];
        }
        this.selectedItems = _.clone(this.selected);

        this.sort_by_key = this.sortByDefault || null;

        Fjord.bus.$on('reload', this._loadItems);

        this.$on('reload', this._loadItems);
        this.$on('refreshSelected', () => {
            this.selectedItems = _.clone(this.selected);
        });
        this.$on('refresh', () => {
            this.$refs.table.$forceUpdate();
        });

        this._loadItems();
    },
    computed: {
        canSort() {
            if (!this.sortable) {
                return false;
            }
            if (this.sortable == 'force') {
                return true;
            }
            return (
                this.sort_by_key == `${this.orderColumn}.asc` ||
                this.sort_by_key == `${this.orderColumn}.desc`
            );
        },
        hasFilter() {
            return Object.keys(this.filter).length !== 0;
        },
        hasSort() {
            return Object.keys(this.sortBy).length !== 0;
        },
        hasTabs() {
            return this.tabs.length > 0;
        }
    },
    methods: {
        sorted(sortedItems) {
            let ids = {};
            let start = this.currentPage * this.perPage - this.perPage;
            if (this.sort_by_key.endsWith('desc')) {
                start = this.total - start - 1;
            }
            for (let i in sortedItems) {
                if (this.sort_by_key.endsWith('desc')) {
                    ids[start - parseInt(i)] = sortedItems[i].id;
                } else {
                    ids[start + parseInt(i)] = sortedItems[i].id;
                }
            }
            this.$emit('sorted', { sortedItems, ids });
        },
        sort(sort) {
            this.sort_by_key = sort;
            this._loadItems();
        },
        newTab(index) {
            this.tab = this.tabs[index];
            this.resetCurrentPage();
            this._loadItems();
        },
        filterChanged(filter) {
            this.filter_scope = filter;
            this.resetCurrentPage();
            this._loadItems();
        },
        sort(key) {
            this.sort_by_key = key;
            this._loadItems();
        },
        unselect(item) {
            for (let i in this.selectedItems) {
                if (this.selectedItems[i].id == item.id) {
                    this.selectedItems.splice(i, 1);
                }
            }
        },
        select(item) {
            if (this.radio) {
                this.selectedItems = [];
                this.selectedItems.push(item);
            } else {
                this.selectedItems.push(item);
            }
        },
        isItemSelected(item) {
            return this.selectedItems.find(model => {
                return model ? model.id == item.id : false;
            })
                ? true
                : false;
        },
        executedActions() {
            this.selectedItems = [];
            this._loadItems();
        },
        async _loadItems() {
            if (!this.loadItems) {
                return;
            }
            this.isBusy = true;

            let payload = {
                tab: this.tab,
                page: this.currentPage,
                perPage: this.perPage,
                search: this.search,
                sort_by: this.sort_by_key,
                filter: this.filter_scope,
                searchKeys: this.searchKeys
            };

            let response = await this.loadItems(payload);

            this.$refs.table.$emit('loaded');

            this.isBusy = false;

            if (!response) return;
            if (!'data' in response) return;
            if (!'count' in response.data) return;

            this.numberOfPages = Math.ceil(response.data.count / this.perPage);
            this.total = response.data.count;
        },
        resetCurrentPage() {
            this.currentPage = 1;
        },
        doSearch(query) {
            this.search = query;
            this.resetCurrentPage();
            this._loadItems();
        },
        goToPage(page) {
            this.currentPage = page;
            this._loadItems();
        },
        linkGen(pageNum) {
            return { path: `#${pageNum}` };
        }
    }
};
</script>

<style lang="scss">
@import '@fj-sass/_variables';
.fj-index-table {
    .nav-tabs {
        margin-left: 0;
        margin-right: 0;
    }

    .table-responsive {
        width: auto;
        margin-left: -$card-spacer-x;
        margin-right: -$card-spacer-x;
        min-width: calc(100% + #{2 * $card-spacer-x});
        @media (max-width: map-get($grid-breakpoints, $nav-breakpoint-mobile)) {
            margin-left: -$page-margin-sm;
            margin-right: -$page-margin-sm;
            min-width: unset;
        }
    }

    .paginated {
        table.b-table {
            tbody {
                tr:last-child {
                    td {
                        border-bottom: 1px solid $gray-300;
                    }
                }
            }
        }
    }

    table.b-table {
        &[aria-busy='true'] {
            opacity: 0.6;
        }

        thead th {
            border-top: none;
            border-bottom: 1px solid $gray-300;

            &:first-child {
                padding-left: $card-spacer-x;
                @media (max-width: map-get($grid-breakpoints, $nav-breakpoint-mobile)) {
                    padding-left: $page-margin-sm;
                }
            }

            &:last-child {
                padding-right: $card-spacer-x;
                @media (max-width: map-get($grid-breakpoints, $nav-breakpoint-mobile)) {
                    padding-right: $page-margin-sm;
                }
            }
        }

        tbody {
            td {
                vertical-align: middle;
                > div {
                    white-space: nowrap;
                }

                &.pointer {
                    cursor: pointer;
                }

                &:first-child {
                    padding-left: $card-spacer-x;
                    @media (max-width: map-get($grid-breakpoints, $nav-breakpoint-mobile)) {
                        padding-left: $page-margin-sm;
                    }
                }

                &:last-child {
                    padding-right: $card-spacer-x;
                    @media (max-width: map-get($grid-breakpoints, $nav-breakpoint-mobile)) {
                        padding-right: $page-margin-sm;
                    }
                }
            }
        }

        .b-table-busy-slot {
            height: 300px;

            td {
                vertical-align: middle;
            }
        }
    }

    &__form {
        display: flex;
        flex-direction: row;
        align-items: stretch;

        .input-group {
            flex: 1;
        }
    }
    &__index-indicator {
        position: absolute;
        margin-top: -10px;
        right: 20px;
    }
}

.btn-brl-none {
    button {
        border-top-left-radius: 0;
        border-bottom-left-radius: 0;
    }
}
.btn-br-none {
    button {
        border-radius: 0;
    }
}

.fj-index-checkbox__group {
    width: auto;
    margin-top: -7px;
    transform: translateX(-0.75rem);

    .input-group-prepend {
        &:first-child {
            .input-group-text {
                padding-left: calc(0.75rem - 1px);
                //width: 37px;
                background: transparent;
            }
        }
    }
}
.fj-index-table-pagination {
    margin-top: -8px;
}
</style>
