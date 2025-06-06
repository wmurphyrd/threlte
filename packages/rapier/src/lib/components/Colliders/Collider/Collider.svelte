<script
  lang="ts"
  generics="TShape extends Shape, TMassDef extends MassDef"
>
  import {
    ActiveCollisionTypes,
    CoefficientCombineRule,
    ColliderDesc
  } from '@dimforge/rapier3d-compat'
  import { createParentObject3DContext, useParentObject3D, useTask, watch } from '@threlte/core'
  import { onDestroy, onMount, tick } from 'svelte'
  import { Object3D, Quaternion, Vector3 } from 'three'
  import { useCollisionGroups } from '../../../hooks/useCollisionGroups'
  import { useRapier } from '../../../hooks/useRapier'
  import { useRigidBody } from '../../../hooks/useRigidBody'
  import { applyColliderActiveEvents } from '../../../lib/applyColliderActiveEvents'
  import { eulerToQuaternion } from '../../../lib/eulerToQuaternion'
  import { getWorldPosition, getWorldQuaternion } from '../../../lib/getWorldTransforms'
  import { useParentRigidbodyObject } from '../../../lib/rigidBodyObjectContext'
  import { scaleColliderArgs } from '../../../lib/scaleColliderArgs'
  import { useCreateEvent } from '../../../lib/useCreateEvent'
  import type { ColliderProps, MassDef, Shape } from './types'

  let {
    shape,
    args,
    type,
    restitution,
    restitutionCombineRule,
    friction,
    frictionCombineRule,
    sensor,
    contactForceEventThreshold,
    density,
    mass,
    centerOfMass,
    principalAngularInertia,
    angularInertiaLocalFrame,
    collider = $bindable(),
    oncreate,
    oncollisionenter,
    oncollisionexit,
    oncontact,
    onsensorenter,
    onsensorexit,
    children
  }: ColliderProps<TShape, TMassDef> = $props()

  const object = new Object3D()

  const { updateRef } = useCreateEvent(oncreate)
  const rigidBody = useRigidBody()
  const parentRigidBodyObject = useParentRigidbodyObject()
  const hasRigidBodyParent = !!rigidBody

  const rapierContext = useRapier()
  const { world } = rapierContext

  const collisionGroups = useCollisionGroups()

  const events = {
    oncollisionenter,
    oncollisionexit,
    oncontact,
    onsensorenter,
    onsensorexit
  }

  /**
   * Actual collider setup happens onMount as only then
   * the transforms are finished.
   */
  onMount(async () => {
    await tick()

    const scale = object.getWorldScale(new Vector3())

    const scaledArgs = scaleColliderArgs(shape, args, scale)

    // @ts-expect-error
    const colliderDesc = ColliderDesc[shape](...scaledArgs) as ColliderDesc

    collider = world.createCollider(colliderDesc, rigidBody)
    collider.setActiveCollisionTypes(ActiveCollisionTypes.ALL)
    updateRef(collider)

    /**
     * Add collider to context
     */
    rapierContext.addColliderToContext(collider, object, events)

    /**
     * For use in conjunction with component <CollisionGroups>
     */
    collisionGroups.registerColliders([collider])

    if (hasRigidBodyParent) {
      const rigidBodyWorldPos = new Vector3()
      const rigidBodyWorldQuatInversed = new Quaternion()

      parentRigidBodyObject?.getWorldPosition(rigidBodyWorldPos)
      parentRigidBodyObject?.getWorldQuaternion(rigidBodyWorldQuatInversed)
      rigidBodyWorldQuatInversed.invert()

      const worldPosition = object.getWorldPosition(new Vector3()).sub(rigidBodyWorldPos)
      const worldRotation = object
        .getWorldQuaternion(new Quaternion())
        .premultiply(rigidBodyWorldQuatInversed)

      collider.setTranslationWrtParent(worldPosition)
      collider.setRotationWrtParent(worldRotation)
    } else {
      collider.setTranslation(object.getWorldPosition(new Vector3()))
      collider.setRotation(object.getWorldQuaternion(new Quaternion()))
    }
  })

  $effect.pre(() => {
    collider?.setRestitution(restitution ?? 0)
  })
  $effect.pre(() => {
    collider?.setRestitutionCombineRule(restitutionCombineRule ?? CoefficientCombineRule.Average)
  })
  $effect.pre(() => {
    collider?.setFriction(friction ?? 0.7)
  })
  $effect.pre(() => {
    collider?.setFrictionCombineRule(frictionCombineRule ?? CoefficientCombineRule.Average)
  })
  $effect.pre(() => collider?.setSensor(sensor ?? false))
  $effect.pre(() => collider?.setContactForceEventThreshold(contactForceEventThreshold ?? 0))
  $effect.pre(() => {
    if (density !== undefined) collider?.setDensity(density)
  })

  $effect.pre(() => {
    if (collider && mass) {
      if (centerOfMass && principalAngularInertia && angularInertiaLocalFrame) {
        collider.setMassProperties(
          mass,
          { x: centerOfMass[0], y: centerOfMass[1], z: centerOfMass[2] },
          {
            x: principalAngularInertia[0],
            y: principalAngularInertia[1],
            z: principalAngularInertia[2]
          },
          eulerToQuaternion(angularInertiaLocalFrame)
        )
      } else {
        collider.setMass(mass)
      }
    }
  })

  $effect.pre(() => {
    if (collider) {
      applyColliderActiveEvents(collider, events, rigidBody?.userData?.events)
    }
  })

  export const refresh = () => {
    if (!collider) return
    collider.setTranslation(getWorldPosition(object))
    collider.setRotation(getWorldQuaternion(object))
  }

  /**
   * If the Collider isAttached (i.e. NOT child of a RigidBody), update the
   * transforms on every frame.
   */
  const { start, stop } = useTask(
    () => {
      refresh()
    },
    {
      autoStart: !hasRigidBodyParent && type === 'dynamic'
    }
  )

  $effect.pre(() => {
    if (!hasRigidBodyParent && type === 'dynamic') start()
    else stop()
  })

  /**
   * Cleanup
   */
  onDestroy(() => {
    if (!collider) return
    rapierContext.removeColliderFromContext(collider)
    collisionGroups.removeColliders([collider])
    world.removeCollider(collider, true)
    collider = undefined
  })

  const parent3DObject = useParentObject3D()
  createParentObject3DContext(object)
  watch(parent3DObject, (parent) => {
    parent?.add(object)
    return () => {
      parent?.remove(object)
    }
  })
</script>

{@render children?.({ collider })}
