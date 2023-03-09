# pdfcontroller
https://www.youtube.com/watch?v=q7b23612pSc


## Description
This create a `PdfDocument` resource type.

## Re doing the tuto

### Prerequisite

You must install `kubebuilder` (https://github.com/kubernetes-sigs/kubebuilder)

```bash
git clone https://github.com/kubernetes-sigs/kubebuilder.git
cd kubebuilder
make build
sudo mv bin/kubebuilder /usr/local/bin/kubebuilder
```

Create a k8s cluster:

```bash
kind create cluster -n my-cluster
```

### Create environment
```bash
mkdir -p pdfcontroller
cd pdfcontroller
go mod init k8s.startkubernetes.com/v1
kubebuilder init
kubebuilder create api --group k8s.startkubernetes.com --version v1 --kind PdfDocument
```

### Write code

#### Types

Open file `api/v1/pdfdocument_types.go` and add the following fields 

```golang
// PdfDocumentSpec defines the desired state of PdfDocument
type PdfDocumentSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of PdfDocument. Edit pdfdocument_types.go to remove/update
	DocumentName string `json:"documentName,omitempty"`
	Text         string `json:"text,omitempty"`
}
```

#### Controller

1. Open file `controllers/pdfdocument_controller.go`
2. Add imports

```golang
// batchv1 "k8s.io/api/batch/v1"
// corev1 "k8s.io/api/core/v1"
// metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
```

3. Write following code

```golang
func (r *PdfDocumentReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := log.FromContext(ctx)

	var pdfDoc k8sstartkubernetescomv1.PdfDocument
	if err := r.Get(ctx, req.NamespacedName, &pdfDoc); err != nil {
		log.Error(err, "unable to fetch PdfDocument")

		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	jobSpec, err := r.createJob(pdfDoc)
	if err != nil {
		log.Error(err, "fail to create job spec")

		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	if err := r.Create(ctx, &jobSpec); err != nil {
		log.Error(err, "unable to create job")
	}

	return ctrl.Result{}, nil
}

func (r *PdfDocumentReconciler) createJob(pdfDoc k8sstartkubernetescomv1.PdfDocument) (batchv1.Job, error) {
	image := "knsit/pandoc"
	base64text := base64.StdEncoding.EncodeToString([]byte(pdfDoc.Spec.Text))

	j := batchv1.Job{
		TypeMeta: metav1.TypeMeta{APIVersion: batchv1.SchemeGroupVersion.String(), Kind: "Job"},
		ObjectMeta: metav1.ObjectMeta{
			Name:      pdfDoc.Name + "-job",
			Namespace: pdfDoc.Namespace,
		},
		Spec: batchv1.JobSpec{
			Template: corev1.PodTemplateSpec{
				Spec: corev1.PodSpec{
					RestartPolicy: corev1.RestartPolicyOnFailure,
					InitContainers: []corev1.Container{
						{
							Name:    "store-to-md",
							Image:   "alpine",
							Command: []string{"/bin/sh"},
							Args:    []string{"-c", fmt.Sprintf("echo %s | base64 -d >> /data/text.md", base64text)},
							VolumeMounts: []corev1.VolumeMount{
								{
									Name:      "data-volume",
									MountPath: "/data",
								},
							},
						},
						{
							Name:    "convert",
							Image:   image,
							Command: []string{"sh", "-c"},
							Args:    []string{fmt.Sprintf("pandoc -s -o /data/%s.pdf /data/text.md", pdfDoc.Spec.DocumentName)},
							VolumeMounts: []corev1.VolumeMount{
								{
									Name:      "data-volume",
									MountPath: "/data",
								},
							},
						},
					},
					Containers: []corev1.Container{
						{
							Name:    "main",
							Image:   "alpine",
							Command: []string{"sh", "-c", "sleep 3600"},
							VolumeMounts: []corev1.VolumeMount{
								{
									Name:      "data-volume",
									MountPath: "/data",
								},
							},
						},
					},
					Volumes: []corev1.Volume{
						{
							Name: "data-volume",
							VolumeSource: corev1.VolumeSource{
								EmptyDir: &corev1.EmptyDirVolumeSource{},
							},
						},
					},
				},
			},
		},
	}
	return j, nil
}
```

### make the project

```bash
make
```

this will generate `config/crd/bases/k8s.startkubernetes.com.my.domain_pdfdocuments.yaml`

```bash
kubectl apply -f config/crd/bases/k8s.startkubernetes.com.my.domain_pdfdocuments.yaml

# check
kubectl get crd
```

### Test it

Launch the stack:

```bash
make run
```

Create a PDF:

```bash
kubectl apply -f ./pdf.yaml

# check
kubectl get pdfdocument
kubectl get jobs
kubectl get pod

# Wait a little bit so the pod is ready and finished his job.

# Get the PDF (replace xxxxx by the hash of the pod)
kubectl cp my-document-job-xxxxx:/data/my-text.pdf ${PWD}/my-text.pdf
```

### Finish

You can delete the cluster:

```bash
    kind delete cluster -n my-cluster
```

## Getting Started
You’ll need a Kubernetes cluster to run against. You can use [KIND](https://sigs.k8s.io/kind) to get a local cluster for testing, or run against a remote cluster.
**Note:** Your controller will automatically use the current context in your kubeconfig file (i.e. whatever cluster `kubectl cluster-info` shows).

### Running on the cluster
1. Install Instances of Custom Resources:

```sh
kubectl apply -f config/samples/
```

2. Build and push your image to the location specified by `IMG`:

```sh
make docker-build docker-push IMG=<some-registry>/pdfcontroller:tag
```

3. Deploy the controller to the cluster with the image specified by `IMG`:

```sh
make deploy IMG=<some-registry>/pdfcontroller:tag
```

### Uninstall CRDs
To delete the CRDs from the cluster:

```sh
make uninstall
```

### Undeploy controller
UnDeploy the controller from the cluster:

```sh
make undeploy
```

## Contributing
// TODO(user): Add detailed information on how you would like others to contribute to this project

### How it works
This project aims to follow the Kubernetes [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

It uses [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/),
which provide a reconcile function responsible for synchronizing resources until the desired state is reached on the cluster.

### Test It Out
1. Install the CRDs into the cluster:

```sh
make install
```

2. Run your controller (this will run in the foreground, so switch to a new terminal if you want to leave it running):

```sh
make run
```

**NOTE:** You can also run this in one step by running: `make install run`

### Modifying the API definitions
If you are editing the API definitions, generate the manifests such as CRs or CRDs using:

```sh
make manifests
```

**NOTE:** Run `make --help` for more information on all potential `make` targets

More information can be found via the [Kubebuilder Documentation](https://book.kubebuilder.io/introduction.html)

## License

Copyright 2023.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

