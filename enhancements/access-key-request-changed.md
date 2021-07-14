  rules:
  - operations:
    - UPDATE
    apiGroups:
    - engine.kubevault.com
    apiVersions:
    - "*"
    resources:
    - gcpaccesskeyrequests
    - gcpaccesskeyrequests/status
  failurePolicy: Fail


	if req.Operation != admission.Update ||
		!(req.Subresource == "" || req.Subresource == "status") ||
		req.Kind.Group != api.SchemeGroupVersion.Group ||
		req.Kind.Kind != api.ResourceKindGCPAccessKeyRequest {
		status.Allowed = true
		return status
	}


	if isApprovedOrDenied {
		1. No spec changed between old vs new
		2. status.phase changed old vs new
		3. in new object, status.condition condition type == status.phase


Phase: WaitingForApproval
[
]

Phase: Denied
[
	{Type: Denied, Status: True}
]

Phase: Approved
[
	{Type: Approved, Status: True}
]

Phase: Approved
[
	{Type: Approved, Status: True}
	{Type: Available, Status: True}
]
secret: xyz

Phase: Approved
[
	{Type: Approved, Status: True}
	{Type: Failure, Status: True, }
]


		// once request is approved or denied, .spec can not be changed
		diff := meta_util.Diff(oldGcpAKReq.Spec, gcpAKReq.Spec)
		if diff != "" {
			return hookapi.StatusBadRequest(errors.Errorf("once request is approved or denied, .spec can not be changed. Diff: %s", diff))
		}
	}


	// https://github.com/kubedb/mongodb/blob/master/pkg/admission/validator.go#L302-L349
