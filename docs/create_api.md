```bash
[kauri@kdev memcached-operator]$ kubebuilder create api --group cache --version v1alpha1 --kind Memcached
INFO Create Resource [y/n]                        
y
INFO Create Controller [y/n]                      
y
INFO Writing kustomize manifests for you to edit... 
INFO Writing scaffold for you to edit...          
INFO api/v1alpha1/memcached_types.go              
INFO api/v1alpha1/groupversion_info.go            
INFO internal/controller/suite_test.go            
INFO internal/controller/memcached_controller.go  
INFO internal/controller/memcached_controller_test.go 
INFO Update dependencies:
$ go mod tidy           
INFO Running make:
$ make generate                
mkdir -p /home/kauri/go/memcached-operator/bin
Downloading sigs.k8s.io/controller-tools/cmd/controller-gen@v0.18.0
go: sigs.k8s.io/controller-tools@v0.18.0 requires go >= 1.24.0; switching to go1.24.4
/home/kauri/go/memcached-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
Next: implement your new API and generate the manifests (e.g. CRDs,CRs) with:
$ make manifests
[kauri@kdev memcached-operator]$ 
```
