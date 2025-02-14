<template>
	<div class="user-role-list" :class="{ manage: canManage }">
		<div
			v-for="role of roles"
			:key="role.id"
			class="user-role-chip"
			:style="{ color: ConvertIntColorToHex(role.color) }"
		>
			<span>{{ role.name }}</span>

			<Icon icon="remove" class="role-interact-remove" @click="setRole(role, 'REMOVE')" />
		</div>

		<!-- Add Role -->
		<div v-if="canManage" class="role-interact-add" @click="openRoleMenu">
			<Icon icon="plus" />
		</div>

		<!-- Role Selector -->
		<div v-if="roleSelectorOpen">
			<div ref="roleSelector" class="role-selector">
				<div
					v-for="role of availableRoles.values()"
					:key="role.id"
					class="grantable-role"
					@click="setRole(role, 'ADD')"
				>
					<span :style="{ color: ConvertIntColorToHex(role.color) }">{{ role.name }}</span>
				</div>
			</div>
		</div>
	</div>
</template>

<script setup lang="ts">
import { UpdateUserRoles } from "@/assets/gql/mutation/UserRoles";
import { useActorStore } from "@/store/actor";
import { useStore } from "@/store/main";
import { Common } from "@/structures/Common";
import { Permissions, Role } from "@/structures/Role";
import { User } from "@/structures/User";
import { ConvertIntColorToHex } from "@/structures/util/Color";
import { getVirtualElement } from "@/structures/util/VirtualElement";
import { useMutation } from "@vue/apollo-composable";
import { storeToRefs } from "pinia";
import { computed, ref } from "vue";
import { createPopper } from "@popperjs/core";
import Icon from "./Icon.vue";
import { onClickOutside } from "@vueuse/core";

const props = defineProps<{
	user: User;
}>();

const roles = computed(() => (User.GetRoles(props.user) ?? []).filter((r) => !r.invisible));

const { roles: storeRoles } = storeToRefs(useStore());
const actor = useActorStore();
const canManage = computed(
	() => actor.hasPermission(Permissions.ManageRoles) && actor.hasPermission(Permissions.ManageUsers),
);

const availableRoles = ref([] as Role[]);

const setRole = (role: Role, action: Common.ListItemAction) => {
	const { onError, onDone, mutate } = useMutation<UpdateUserRoles.Result, UpdateUserRoles.Variables>(
		UpdateUserRoles,
		{
			variables: {
				user_id: props.user.id,
				role_id: role.id,
				action: action,
			},
		},
	);

	mutate();

	onError((err) => {
		actor.showErrorModal(err);
	});
	onDone(() => {
		roleSelectorOpen.value = false;
	});
};

const roleSelector = ref<HTMLElement | null>(null);
const roleSelectorOpen = ref(false);

const openRoleMenu = (ev: MouseEvent) => {
	roleSelectorOpen.value = !roleSelectorOpen.value;

	availableRoles.value = Array.from(storeRoles.value.values()).filter((r) => actor.mayEditRole(r) && !r.invisible);

	if (!roleSelector.value) {
		return;
	}

	const ve = getVirtualElement(ev);
	createPopper(ve, roleSelector.value);
};

onClickOutside(roleSelector, () => {
	roleSelectorOpen.value = false;
});
</script>

<style scoped lang="scss">
@import "@scss/themes.scss";

.user-role-list {
	display: flex;
	flex-wrap: wrap;
	gap: 0.25em;
	margin-right: 0.25em;
	margin-left: 0.25em;

	> .role-interact-add {
		cursor: pointer;
		padding: 0.5em;
		border-radius: 0.25rem;
	}

	@include themify() {
		> .user-role-chip {
			background-color: lighten(themed("backgroundColor"), 2);
		}

		> .user-role-chip > .role-interact-remove {
			color: themed("warning");
		}

		> .role-interact-add {
			background-color: mix(themed("backgroundColor"), themed("primary"), 50%);
		}

		.role-selector {
			background-color: lighten(themed("backgroundColor"), 2);

			> .grantable-role {
				&:hover {
					background-color: lighten(themed("backgroundColor"), 5);
				}
			}
		}
	}

	.user-role-chip {
		display: grid;
		align-items: center;
		column-gap: 0.25em;
		grid-template-columns: auto;

		padding: 0.5em;
		border-radius: 0.25rem;
		max-width: 12em;
		overflow: hidden;
		white-space: nowrap;
		text-overflow: ellipsis;

		> span {
			user-select: none;
		}

		> .role-interact-remove {
			display: none;
			cursor: pointer;
		}
	}

	&.manage {
		> .user-role-chip {
			grid-template-columns: auto auto;

			> .role-interact-remove {
				display: block;
			}
		}
	}

	.role-selector {
		position: absolute;
		z-index: 1;
		display: grid;
		gap: 0.5em;
		border-radius: 0.25rem;
		max-height: 12em;
		overflow: auto;

		> .grantable-role {
			cursor: pointer;
			padding: 0.5em;
			border-radius: 0.25rem;
		}
	}
}
</style>
