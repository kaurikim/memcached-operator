
# 리소스 관찰 (Watching resources)

이 페이지에서는 컨트롤러가 리소스를 어떻게 관찰(watch)하고, 그에 따라 리콘실(reconcile) 루프가 어떻게 트리거되는지 설명합니다.

기본 `controllers/memcached_controller.go` 파일을 살펴보세요:

```bash
cat controllers/memcached_controller.go
```

기본 생성된 컨트롤러는 `kind: Memcached` 객체가 **생성**, **수정**, **삭제**될 때마다 리콘실러가 실행되도록 하는 추가 로직이 필요합니다. 또한, 각 Memcached 인스턴스가 소유한 Deployment에 대해서도 동일하게 **생성**, **수정**, **삭제** 이벤트가 발생할 때마다 리콘실러가 호출되길 원합니다. 이를 위해 컨트롤러의 `SetupWithManager` 메서드를 수정합니다.

아래 예시 코드를 `controllers/memcached_controller.go`에 덮어쓰세요:

```go
package controllers

import (
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    "reflect"
    "time"
    "context"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    ctrllog "sigs.k8s.io/controller-runtime/pkg/log"

    cachev1alpha1 "github.com/example/memcached-operator/api/v1alpha1"
)

// MemcachedReconciler는 Memcached 객체를 리콘실(reconcile) 합니다.
type MemcachedReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// 아래 RBAC 주석은 컨트롤러가 필요한 권한을 선언합니다.
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/finalizers,verbs=update
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch

// Reconcile는 메인 리콘실 루프의 일부로, 
// 현재 클러스터 상태를 원하는 상태에 가깝게 만듭니다.
// 더 자세한 내용은 다음을 참고하세요:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.8.3/pkg/reconcile
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrllog.FromContext(ctx)

    // 1) Memcached 인스턴스를 가져옵니다.
    memcached := &cachev1alpha1.Memcached{}
    err := r.Get(ctx, req.NamespacedName, memcached)
    if err != nil {
        if errors.IsNotFound(err) {
            // 요청된 객체가 없으면 (삭제된 경우), 무시하고 재큐 하지 않습니다.
            log.Info("Memcached 리소스를 찾을 수 없습니다. 삭제된 것으로 간주합니다.")
            return ctrl.Result{}, nil
        }
        log.Error(err, "Memcached를 가져오지 못했습니다.")
        return ctrl.Result{}, err
    }

    // 2) Deployment가 존재하는지 확인하고, 없으면 새로 생성합니다.
    found := &appsv1.Deployment{}
    err = r.Get(ctx, types.NamespacedName{Name: memcached.Name, Namespace: memcached.Namespace}, found)
    if err != nil && errors.IsNotFound(err) {
        dep := r.deploymentForMemcached(memcached)
        log.Info("새 Deployment 생성", "Namespace", dep.Namespace, "Name", dep.Name)
        err = r.Create(ctx, dep)
        if err != nil {
            log.Error(err, "Deployment 생성 실패", "Namespace", dep.Namespace, "Name", dep.Name)
            return ctrl.Result{}, err
        }
        // 생성 후 재큐하여 상태를 다시 확인합니다.
        return ctrl.Result{Requeue: true}, nil
    } else if err != nil {
        log.Error(err, "Deployment 조회 실패")
        return ctrl.Result{}, err
    }

    // 3) 스펙과 동일한 복제 수가 되도록 업데이트합니다.
    size := memcached.Spec.Size
    if *found.Spec.Replicas != size {
        found.Spec.Replicas = &size
        err = r.Update(ctx, found)
        if err != nil {
            log.Error(err, "Deployment 업데이트 실패", "Namespace", found.Namespace, "Name", found.Name)
            return ctrl.Result{}, err
        }
        // 1분 후 재큐하여 다음 업데이트를 처리할 시간을 줍니다.
        return ctrl.Result{RequeueAfter: time.Minute}, nil
    }

    // 4) Pod 목록을 조회하여 상태에 반영합니다.
    podList := &corev1.PodList{}
    listOpts := []client.ListOption{
        client.InNamespace(memcached.Namespace),
        client.MatchingLabels(labelsForMemcached(memcached.Name)),
    }
    if err = r.List(ctx, podList, listOpts...); err != nil {
        log.Error(err, "Pod 목록 조회 실패", "Namespace", memcached.Namespace, "Memcached", memcached.Name)
        return ctrl.Result{}, err
    }
    podNames := getPodNames(podList.Items)

    if !reflect.DeepEqual(podNames, memcached.Status.Nodes) {
        memcached.Status.Nodes = podNames
        err := r.Status().Update(ctx, memcached)
        if err != nil {
            log.Error(err, "Memcached 상태 업데이트 실패")
            return ctrl.Result{}, err
        }
    }

    return ctrl.Result{}, nil
}

// deploymentForMemcached는 Memcached 인스턴스용 Deployment 객체를 생성합니다.
func (r *MemcachedReconciler) deploymentForMemcached(m *cachev1alpha1.Memcached) *appsv1.Deployment {
    ls := labelsForMemcached(m.Name)
    replicas := m.Spec.Size
    dep := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      m.Name,
            Namespace: m.Namespace,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &replicas,
            Selector: &metav1.LabelSelector{MatchLabels: ls},
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{Labels: ls},
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{{
                        Name:  "memcached",
                        Image: "memcached:1.4.36-alpine",
                        Command: []string{"memcached", "-m=64", "-o", "modern", "-v"},
                        Ports: []corev1.ContainerPort{{
                            ContainerPort: 11211,
                            Name:          "memcached",
                        }},
                    }},
                },
            },
        },
    }
    // Memcached 인스턴스를 오너이자 컨트롤러로 설정합니다.
    ctrl.SetControllerReference(m, dep, r.Scheme)
    return dep
}

// labelsForMemcached는 Memcached CR 이름에 대한 레이블을 반환합니다.
func labelsForMemcached(name string) map[string]string {
    return map[string]string{"app": "memcached", "memcached_cr": name}
}

// getPodNames는 Pods 배열에서 이름만 추출합니다.
func getPodNames(pods []corev1.Pod) []string {
    var podNames []string
    for _, pod := range pods {
        podNames = append(podNames, pod.Name)
    }
    return podNames
}

// SetupWithManager는 Manager에 컨트롤러를 등록합니다.
func (r *MemcachedReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&cachev1alpha1.Memcached{}).    // 주 리소스 지정
        Owns(&appsv1.Deployment{}).          // 2차 리소스 지정
        WithOptions(controller.Options{MaxConcurrentReconciles: 2}).
        Complete(r)
}
```

마지막으로 `go mod tidy`를 실행하여 `go.mod`·`go.sum`을 최신 상태로 정리하세요.

---

아래 다이어그램들은 `SetupWithManager()`를 통해 컨트롤러가 **주 리소스**와 **소유 리소스**를 어떻게 관찰(watch)하는지 보여줍니다.

![다이어그램 1](https://kubebyexample.com/learning-paths/operator-framework/operator-sdk-go/watching-resources#264)
![다이어그램 2](https://kubebyexample.com/learning-paths/operator-framework/operator-sdk-go/watching-resources#265)
![다이어그램 3](https://kubebyexample.com/learning-paths/operator-framework/operator-sdk-go/watching-resources#266)
![다이어그램 4](https://kubebyexample.com/learning-paths/operator-framework/operator-sdk-go/watching-resources#267)

* `For(&cachev1alpha1.Memcached{})`는 Memcached CRD를 주 리소스로 지정합니다. 이 리소스에 대한 **생성/수정/삭제** 이벤트가 발생하면, 해당 Namespace/Name 키를 가진 리콘실 요청이 전송됩니다.
* `Owns(&appsv1.Deployment{})`는 Deployment를 소유 리소스로 지정합니다. Deployment에 대한 **생성/수정/삭제** 이벤트가 발생하면, 컨트롤러는 해당 Deployment의 오너인 Memcached 객체로 매핑된 리콘실 요청을 전송합니다.

