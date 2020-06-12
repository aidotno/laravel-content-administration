<template>
    <fj-base-field :field="field" :model="model">
        <template slot="title-right">
            <b-button
                variant="secondary"
                class="mb-2"
                size="sm"
                @click="addListItem"
            >
                {{
                    __('base.item_add', {
                        item: __('base.item_item', { item: field.title })
                    })
                }}
            </b-button>
        </template>
        <div class="d-flex justify-content-around w-100" v-if="busy">
            <fj-spinner />
        </div>
        <nested-draggable
            v-else
            :children="list"
            :field="field"
            :model="model"
            @end="orderListItems"
            @addItem="addListItem"
            @deleteItem="deleteListItem"
        />
    </fj-base-field>
</template>

<script>
import nestedDraggable from './Nested';
export default {
    name: 'FieldList',
    components: {
        nestedDraggable
    },
    props: {
        /**
         * Field attributes.
         */
        field: {
            required: true,
            type: Object
        },

        /**
         * Model.
         */
        model: {
            required: true,
            type: Object
        }
    },
    data() {
        return {
            busy: false,
            list: [],
            input: [
                {
                    id: 1,
                    title: 'Item 1',
                    order_column: 2,
                    parent_id: 0
                },
                {
                    id: 2,
                    title: 'Item 2',
                    order_column: 1,
                    parent_id: 0
                },
                {
                    id: 3,
                    title: 'Item 3',
                    order_column: 3,
                    parent_id: 0
                },
                {
                    id: 4,
                    title: 'Item 4',
                    order_column: 1,
                    parent_id: 3
                }
            ]
        };
    },
    beforeMount() {
        //this.list = this.unflatten(copy);
    },
    async mounted() {
        this.busy = true;
        //await this.loadItems();
        this.busy = false;
    },
    methods: {
        itemToast(key) {
            this.$bvToast.toast(this.__(key, { item: this.__item() }), {
                variant: 'success'
            });
        },

        async orderListItems() {
            let items = _.map(this.flattenCrud(this.list), item => {
                return {
                    id: item.id,
                    order_column: item.order_column,
                    parent_id: item.parent_id
                };
            });
            let response = await this.sendOrderListItems({ items });
            if (!response) {
                return;
            }

            this.$bvToast.toast(
                this.__('base.item_ordered', { item: this.field.title }),
                {
                    variant: 'success'
                }
            );
        },

        sendOrderListItems(payload) {
            try {
                return axios.put(
                    `${this.field.route_prefix}/list/${this.field.id}/order`,
                    payload
                );
            } catch (e) {
                console.log(e);
            }
        },

        /**
         * Delete list item.
         */
        async deleteListItem(item) {
            let response = await this.sendDeleteListItem(item);
            if (!response) {
                return;
            }
            await this.loadItems();

            this.itemToast('base.item_deleted');
        },

        /**
         * Send delete list item.
         */
        sendDeleteListItem(item) {
            try {
                return axios.delete(
                    `${this.field.route_prefix}/list/${this.field.id}/${item.id}`
                );
            } catch (e) {
                console.log(e);
            }
        },

        /**
         * Load items.
         */
        async loadItems() {
            let response = await this.sendLoadItems();
            if (!response) {
                return (this.busy = false);
            }

            let listItems = this.crud(response.data);
            this.list = this.unflatten(listItems);
        },

        /**
         * Send loat items.
         */
        sendLoadItems(parent) {
            try {
                return axios.get(
                    `${this.field.route_prefix}/list/${this.field.id}`
                );
            } catch (e) {
                console.log(e);
            }
        },

        /**
         * Add list.
         */
        async addListItem(parent) {
            let response = await this.sendAddListItem(parent);
            if (!response) {
                return;
            }
            await this.loadItems();

            this.itemToast('base.item_added');
        },

        __item() {
            return this.__('base.item_item', { item: this.field.title });
        },

        /**
         * Send add list item request.
         */
        sendAddListItem(parent) {
            try {
                return axios.post(
                    `${this.field.route_prefix}/list/${this.field.id}`,
                    { parent_id: parent ? parent.id : null }
                );
            } catch (e) {
                console.log(e);
            }
        },

        /**
         * Flatten node generator.
         * Credits: https://github.com/MrPeak/flatten-tree
         */
        flattenNodeGenerator(node, parent, index, settings, stack) {
            const { itemsKey, idKey } = settings;
            return list => {
                node = settings.initNode(node);
                node[idKey] = node[idKey] || settings.generateUniqueId();

                node.order_column = index;

                list.push(node);

                let nodeItems = node[itemsKey];
                if (nodeItems) {
                    for (let i = 0, len = nodeItems.length; i < len; i++) {
                        stack.push(
                            this.flattenNodeGenerator(
                                node[itemsKey][i],
                                node,
                                i,
                                settings,
                                stack
                            )
                        );
                    }
                }

                if (parent && parent[itemsKey]) {
                    // Records children' id
                    parent[itemsKey][index] = node;
                    node.parent_id = parent[idKey];
                } else {
                    node.parent_id = 0;
                }

                return list;
            };
        },

        /**
         * Flatten tree.
         */
        flatten(tree, cloner) {
            let list = [];
            const stack = [];
            const _tree = tree;
            const settings = {
                initNode: node => node,
                itemsKey: 'children',
                idKey: 'id',
                uniqueIdStart: 1,
                generateUniqueId: () => settings.uniqueIdStart++
            };

            if (Array.isArray(_tree) && _tree.length) {
                // Object Array
                for (let i = 0, len = _tree.length; i < len; i++) {
                    stack.push(
                        this.flattenNodeGenerator(
                            _tree[i],
                            'root', // placeholder
                            i,
                            settings,
                            stack
                        )
                    );
                }
            } else {
                // One object tree
                stack.push(
                    this.flattenNodeGenerator(_tree, 'root', 0, settings, stack)
                );
            }

            while (stack.length) {
                list = stack.shift()(list);
            }

            // cleanup
            list = _.map(list, item => {
                if (!item.parent_id) {
                    item.parent_id = 0;
                }
                return _.omit(item, ['children']);
            });

            return list;
        },

        /**
         * Flatten crud tree.
         */
        flattenCrud(tree) {
            // The flattened tree needs to be converted to a crud array since
            // the flatten method is creating a clone which is getting rid of
            // all crud model objects.
            return this.crud(this.flatten(tree));
        },

        /**
         * Unflatten.
         * https://stackoverflow.com/questions/18017869/build-tree-array-from-flat-array-in-javascript
         */
        unflatten(array, parent, tree) {
            array = _.sortBy(array, item => {
                return item.order_column;
            });

            tree = typeof tree !== 'undefined' ? tree : [];
            parent = typeof parent !== 'undefined' ? parent : { id: 0 };

            if (!parent.hasOwnProperty('children')) {
                parent.children = [];
            }

            var children = _.filter(array, child => {
                return child.parent_id == parent.id;
            });

            if (!_.isEmpty(children)) {
                if (parent.id == 0) {
                    tree = children;
                } else {
                    parent['children'] = children;
                }
                _.each(children, child => {
                    this.unflatten(array, child);
                });
            }

            return tree;
        }
    },
    computed: {
        /**
         * Output.
         */
        output() {
            return this.flatten(this.list);
        },

        /**
         * Unflattened.
         */
        unflattened() {
            return this.unflatten(this.input);
        }
    }
};
</script>