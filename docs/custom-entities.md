# Defining Custom Entities (CRDs)

In Kubernetes, a [Custom Resource Definition (CRD)](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) defines a new kind of resource that extends the Kubernetes API. With KubeOps, you define these CRDs using standard C# classes decorated with specific attributes.

This approach provides strong typing, compile-time checks, and excellent tooling support within your [.NET](https://dotnet.microsoft.com/) development environment.

## The `[KubernetesEntity]` Attribute

The core attribute for defining a CRD is `[KubernetesEntity]` from the `KubeOps.Abstractions.Entities.Attributes` namespace. This attribute marks a C# class as a Kubernetes resource entity.

```csharp
using k8s.Models;
using KubeOps.Abstractions.Entities;
using KubeOps.Abstractions.Entities.Attributes;

[KubernetesEntity(Group = "example.com", ApiVersion = "v1alpha1", Kind = "DemoEntity", PluralName = "demoentities")]
public class V1Alpha1DemoEntity : CustomKubernetesEntity<V1Alpha1DemoEntity.DemoEntitySpec, V1Alpha1DemoEntity.DemoEntityStatus>
{
    // Entity implementation follows
}
```

Key parameters for `[KubernetesEntity]`: 

*   **`Kind`**: (Required) The PascalCase name for your custom resource kind (e.g., `MyDatabase`, `WebAppInstance`). This is how users will refer to your resource in YAML manifests (`kind: MyDatabase`).
*   **`Group`**: (Required) The API group for your resource. This provides namespacing and avoids collisions. It typically follows reverse domain name notation (e.g., `example.com`, `apps.example.org`).
*   **`ApiVersion`**: (Required) The version of your custom resource API (e.g., `v1`, `v1alpha1`, `v1beta1`). This allows you to evolve your API over time.
*   **`PluralName`**: (Optional) The lowercase plural name used when interacting with the resource via `kubectl` or the API (e.g., `mydatabases`, `webappinstances`). If omitted, KubeOps attempts to generate it by adding an 's' to the `Kind`. For non-standard plurals, specify it explicitly.

## Spec and Status

Kubernetes resources typically separate the *desired state* (`spec`) from the *observed state* (`status`). KubeOps follows this convention.

Your entity class should inherit from `CustomKubernetesEntity<TSpec, TStatus>`. This base class provides the standard Kubernetes properties like `ApiVersion`, `Kind`, and `Metadata`, which are automatically populated based on the `[KubernetesEntity]` attribute and Kubernetes API interactions. You only need to define your `TSpec` and optionally `TStatus` types.

**Note:** If your custom resource does not require a dedicated `status` subresource (common for simpler or configuration-like resources), you can inherit from `CustomKubernetesEntity<TSpec>` instead.

```csharp
using k8s.Models;
using KubeOps.Abstractions.Entities;
using KubeOps.Abstractions.Entities.Attributes;
using System.ComponentModel.DataAnnotations; // For validation attributes

[KubernetesEntity(Group = "example.com", ApiVersion = "v1alpha1", Kind = "DemoEntity", PluralName = "demoentities")]
public class V1Alpha1DemoEntity : CustomKubernetesEntity<V1Alpha1DemoEntity.DemoEntitySpec, V1Alpha1DemoEntity.DemoEntityStatus>
{
    /// <summary>
    /// Defines the desired state of the DemoEntity.
    /// </summary>
    public class DemoEntitySpec
    {
        [Required] // Example using .NET Validation Attributes
        [Description("The desired message to be displayed.")]
        public string Message { get; set; } = string.Empty;

        public int Replicas { get; set; } = 1;
    }

    /// <summary>
    /// Describes the observed state of the DemoEntity.
    /// This is typically updated by the controller.
    /// </summary>
    public class DemoEntityStatus
    {
        public string ObservedMessage { get; set; } = string.Empty;
        public string State { get; set; } = "Pending";
        public DateTime? LastUpdated { get; set; }
    }
}
```

*   **`TSpec`**: A class defining the fields that users will set to specify the desired configuration of the resource.
*   **`TStatus`**: A class defining the fields that your operator's controller will update to reflect the current state of the managed resource(s) in the cluster.

## Additional Attributes & Validation

*   **`[Description("...")]`**: Add descriptions to your entity and its properties. These descriptions are included in the generated CRD manifest and are shown when users run `kubectl explain demoentities`.
*   **[.NET Validation Attributes](https://learn.microsoft.com/en-us/dotnet/api/system.componentmodel.dataannotations):** You can use standard .NET validation attributes (like `[Required]`, `[Range]`, `[RegularExpression]`, `[MaxLength]`, etc.) on your `Spec` properties. KubeOps translates these into OpenAPI v3 validation rules within the generated CRD, providing server-side validation via the Kubernetes API server.

## CRD Generation

While you define entities using C# classes, Kubernetes needs the CRD defined in YAML format. KubeOps provides tools to generate this YAML automatically:

1.  **KubeOps.Cli:** The simplest way is using the KubeOps [CLI Tool](./cli.md):
    ```bash
    # Navigate to the directory containing your solution (.sln) or project (.csproj) file
    cd /path/to/your/project

    # Generate CRDs for all entities found in the specified project
    # Adjust the project path as needed
    dotnet kubeops generate crds --project ./MyFirstOperator.Entities/MyFirstOperator.Entities.csproj --output-path ./deploy
    ```
    This command inspects the specified project, finds classes marked with `[KubernetesEntity]`, and generates the corresponding CRD YAML files in the `--output-path`.

2.  **Source Generators (`KubeOps.Generator`):** The `KubeOps.Generator` package (typically included in new projects scaffolded by the CLI) uses [.NET Source Generators](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview). This is the **recommended approach during development**, as it automatically generates CRD YAML during the build process whenever your entity definitions change. Look for generated files in your project's `obj/Generated/KubeOps.Generator/` directory (relative to the project file) after building. You can commit these generated CRDs or use them in deployment scripts.

Typically, you'll rely on the source generator for up-to-date CRDs during development and use the CLI command for specific tasks like exporting CRDs for a CI/CD pipeline or manual inspection.

## Resource Scope

By default, custom resources defined with KubeOps are **Namespaced**. This means instances of your resource must belong to a specific [Kubernetes Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).

To define a **Cluster-scoped** resource (one that exists outside any namespace, like `Nodes` or `ClusterRoles`), add the `[EntityScope(EntityScope.Cluster)]` attribute to your entity class:

```csharp
using k8s.Models;
using KubeOps.Abstractions.Entities;
using KubeOps.Abstractions.Entities.Attributes;
using KubeOps.Abstractions.Entities.Attributes.Validation;

[KubernetesEntity(Group = "example.com", ApiVersion = "v1", Kind = "ClusterData", PluralName = "clusterdata")]
[EntityScope(EntityScope.Cluster)] // Mark as Cluster-scoped
public class V1ClusterData : CustomKubernetesEntity<V1ClusterData.Spec>
{
    public class Spec
    {
        public string GlobalSetting { get; set; } = string.Empty;
    }
    // Cluster-scoped resources typically don't have a Status subresource defined in the same way,
    // though you can still add one if needed.
}
```

The generated CRD will reflect this scope (`scope: Cluster` instead of `scope: Namespaced`).

## Additional Property Attributes

Beyond basic validation, KubeOps offers attributes to fine-tune the generated CRD's OpenAPI schema:

*   **`[ExternalDocs("url")]`**: Links to external documentation for a property.
*   **`[Items("min", "max")]`**: Specifies minimum and maximum item counts for array properties.
*   **`[Length("min", "max")]`**: Specifies minimum and maximum length for string properties.
*   **`[MultipleOf("value")]`**: Requires a numeric property to be a multiple of the specified value.
*   **`[Pattern("regex")]`**: Validates a string property against a regular expression (equivalent to `[RegularExpression]`).
*   **`[RangeMaximum("value", exclusive)]`**: Sets a maximum value for a numeric property (optionally exclusive).
*   **`[RangeMinimum("value", exclusive)]`**: Sets a minimum value for a numeric property (optionally exclusive).

These attributes reside in the `KubeOps.Abstractions.Entities.Attributes.Validation` namespace and directly influence the `validation` section within the generated CRD's OpenAPI v3 schema.

Example:

```csharp
    public class DemoEntitySpec
    {
        [Required]
        [Description("The desired message to be displayed.")]
        [Length(5, 50)] // Message must be between 5 and 50 characters
        public string Message { get; set; } = string.Empty;

        [RangeMinimum(0)] // Replicas cannot be negative
        [RangeMaximum(10)] // Replicas cannot exceed 10
        public int Replicas { get; set; } = 1;

        [Items(1, 5)] // Must have between 1 and 5 tags
        public List<string> Tags { get; set; } = new List<string>();
    }

```

## Example

See a practical example of custom entity definitions in the GitHub repository:
[`examples/Operator/Entities/`](https://github.com/ewassef/dotnet-operator-sdk/tree/main/examples/Operator/Entities/)
