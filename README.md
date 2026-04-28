This repository is the declarative source of truth for the Behavior-Aware                                                                                                                                
  Resilient Kubernetes Platform. ArgoCD reconciles cluster state from this repo.
                                                                                                                                                                                                           
  ## Layout                                                 
                                                                                                                                                                                                           
  - `bootstrap/` — root App-of-Apps and ApplicationSets                                                                                                                                                    
  - `platform/` — platform services (cert-manager, ingress, storage, observability)
  - `apps/` — workload applications (sample-app, future tenants) 
    - `base/` — shared base manifests 
  - `overlays/{primary,secondary}/` — per-cluster Kustomize overlay

