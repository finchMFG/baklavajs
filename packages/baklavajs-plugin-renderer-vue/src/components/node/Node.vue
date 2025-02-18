<template>
    <div :id="data.id" :class="classes" :style="styles">
        <div class="__title" @mousedown.self.stop="startDrag" @contextmenu.self.prevent="openContextMenu">
            <span v-if="!renaming">{{ data.name }}</span>
            <input
                v-else
                type="text"
                class="dark-input"
                v-model="tempName"
                placeholder="Node Name"
                v-click-outside="doneRenaming"
                @keydown.enter="doneRenaming"
            />

            <component
                :is="plugin.components.contextMenu"
                v-model="contextMenu.show"
                :x="contextMenu.x"
                :y="contextMenu.y"
                :items="contextMenu.items"
                @click="onContextMenu"
            ></component>

            <span 
                class="close-icon" 
                @click="closeNode()"
                title="Close Node"
                v-if="plugin.showCloseIcon"
            >
                <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 18 18" fill="white"><path d="M14.53 4.53l-1.06-1.06L9 7.94 4.53 3.47 3.47 4.53 7.94 9l-4.47 4.47 1.06 1.06L9 10.06l4.47 4.47 1.06-1.06L10.06 9z"/></svg> 
            </span>
        </div>

        <div class="__content">
            <!-- Outputs -->
            <div class="__outputs">
                <component
                    :is="plugin.components.nodeInterface"
                    v-for="(output, name) in data.outputInterfaces"
                    :key="output.id"
                    :name="name"
                    :data="output"
                ></component>
            </div>

            <!-- Options -->
            <div class="__options">
                <template v-for="[name, option] in data.options">
                    <component
                        :is="plugin.components.nodeOption"
                        :key="name"
                        :name="name"
                        :option="option"
                        :componentName="option.optionComponent"
                        :node="data"
                        @openSidebar="openSidebar(name)"
                    ></component>

                    <portal
                        :key="'sb_' + name"
                        to="sidebar"
                        v-if="
                            plugin.sidebar.nodeId === data.id &&
                            plugin.sidebar.optionName === name &&
                            option.sidebarComponent
                        "
                    >
                        <component
                            :is="plugin.components.nodeOption"
                            :key="data.id + name"
                            :name="name"
                            :option="option"
                            :componentName="option.sidebarComponent"
                            :node="data"
                        ></component>
                    </portal>
                </template>
            </div>

            <!-- Inputs -->
            <div class="__inputs">
                <component
                    :is="plugin.components.nodeInterface"
                    v-for="(input, name) in data.inputInterfaces"
                    :key="input.id"
                    :name="name"
                    :data="input"
                ></component>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Inject } from "vue-property-decorator";

// @ts-ignore
import ClickOutside from "v-click-outside";

import { ViewPlugin } from "../../viewPlugin";
import { IViewNode } from "../../../types";
import { sanitizeName } from "../../utility/cssNames";

interface IPosition {
    x: number;
    y: number;
}

@Component({
    directives: {
        ClickOutside: ClickOutside.directive,
    },
})
export default class NodeView extends Vue {
    @Prop({ type: Object })
    data!: IViewNode;

    @Prop({ type: Boolean, default: false })
    selected!: boolean;

    @Inject("plugin")
    plugin!: ViewPlugin;

    draggingStartPosition: IPosition | null = null;
    draggingStartPoint: IPosition | null = null;
    renaming = false;
    tempName = "";

    contextMenu = {
        show: false,
        x: 0,
        y: 0,
        items: [
            { value: "rename", label: "Rename" },
            { value: "delete", label: "Delete" },
        ],
    };

    get classes() {
        return {
            "node": true,
            "--selected": this.selected,
            "--dragging": !!this.draggingStartPoint,
            "--two-column": !!this.data.twoColumn,
            [`--type-${sanitizeName(this.data.type)}`]: true,
        };
    }

    get styles() {
        return {
            top: `${this.data.position.y}px`,
            left: `${this.data.position.x}px`,
            width: `${this.data.width}px`,
        };
    }

    mounted() {
        this.data.events.addInterface.addListener(this, () => this.update());
        this.data.events.removeInterface.addListener(this, () => this.update());
        this.data.events.addOption.addListener(this, () => this.update());
        this.data.events.removeOption.addListener(this, () => this.update());
        this.plugin.hooks.renderNode.execute(this);
    }

    updated() {
        this.plugin.hooks.renderNode.execute(this);
    }

    beforeDestroy() {
        this.data.events.addInterface.removeListener(this);
        this.data.events.removeInterface.removeListener(this);
        this.data.events.addOption.removeListener(this);
        this.data.events.removeOption.removeListener(this);
    }

    update() {
        this.$forceUpdate();
    }

    startDrag(ev: MouseEvent) {
        this.draggingStartPoint = {
            x: ev.screenX,
            y: ev.screenY,
        };
        this.draggingStartPosition = {
            x: this.data.position.x,
            y: this.data.position.y,
        };
        document.addEventListener("mousemove", this.handleMove);
        document.addEventListener("mouseup", this.stopDrag);
        this.select();
    }

    select() {
        this.$emit("select", this);
    }

    stopDrag() {
        this.draggingStartPoint = null;
        this.draggingStartPosition = null;
        document.removeEventListener("mousemove", this.handleMove);
        document.removeEventListener("mouseup", this.stopDrag);
    }

    handleMove(ev: MouseEvent) {
        if (this.draggingStartPoint) {
            const dx = ev.screenX - this.draggingStartPoint.x;
            const dy = ev.screenY - this.draggingStartPoint.y;
            this.data.position.x = this.draggingStartPosition!.x + dx / this.plugin.scaling;
            this.data.position.y = this.draggingStartPosition!.y + dy / this.plugin.scaling;
        }
    }

    openContextMenu(ev: MouseEvent) {
        this.contextMenu.show = true;
        this.contextMenu.x = ev.offsetX;
        this.contextMenu.y = ev.offsetY;
    }

    onContextMenu(action: string) {
        switch (action) {
            case "delete":
                this.plugin.editor.removeNode(this.data);
                break;
            case "rename":
                this.tempName = this.data.name;
                this.renaming = true;
        }
    }

    doneRenaming() {
        this.data.name = this.tempName;
        this.renaming = false;
    }

    openSidebar(optionName: string) {
        this.plugin.sidebar.nodeId = this.data.id;
        this.plugin.sidebar.optionName = optionName;
        this.plugin.sidebar.visible = true;
    }

    closeNode() {
        this.plugin.editor.removeNode(this.data);
    }
}
</script>
