# Adding custom operators

This tutorial shows how to register custom operators in the uTensor code generation tools using the plugin system. We will start by defining a custom Layer in Keras using existing Tensorflow operators for clarity and brevity, however extending the concepts in this tutorial to map fully custom framework operators to the uTensor form should be straightforward.  

## The operator lifecycle

![utensor-cli-components](images/utensor-cli-components.drawio.svg)

The operator lifecycle loosely falls into 3 categories, the Frontend, the Transformer, and the Backend. 

- The goal of the `Frontend` is to lift tensor and operator graphs from the various frameworks (Tensorflow, ONNX, etc.) into the uTensor Intermediate Representation (IR). At this stage, the IR makes no assumptions on whether or not a particular operator is "supported", so for the sake of this tutorial is therefore largely ignored. Finally, at the end of this stage and right before the next is the `Legalizer`. This legalization simply ensures that the IR can be processed in the most generic form. For example, we might simply rename `Dense` or `DenseLayer` to `FullyConnectedOperator`.
- The goal of the `Transformer`, as its name implies, is to transform uTensor IR graphs. For example, generating memory plans, rewriting graphs, and generally optimizing inference. Again, this is predominantly operator agnostic so is mostly ignored in this tutorial. 
- The goal of the `Backend` is to do the final lowering of an optimized IR into either code or binary forms. At the end of the transformation pipeline, the final graph is then *lowered* into a target `Backend`. This lowering process is Backend specific and allows the backend to inject additional attributes into the IR before finalizing a strategy for mapping IR components to their respective backend handlers. For example, in the uTensor backened we can inject namespace information to prioritize CMSIS handlers over reference floating point depending on if the operator in question is quantized. Next, the uTensor backend can use these respective handlers to compose a set of code snippets, which ultimately becomes the output model code.


This means there are a total of 4 locations where we might want to register our custom operator, but some of them may be optional based on use-case:

1. Frontend parsing
2. Legalization
3. Backend Lowering
4. Backend Component

For `utensor` backend, there is one more abstraction in the Backend Component for handling code snippet composition:
1. Backend Snippet, code generator in `utensor` is snippet-based.

## Adding custom operators workflow

To keep things from being tied to a particular frontend, we will work with an op that is already present in TFLite, but targets a custom runtime operator in uTensor. Suppose the runtime interface for this operator is fixed and looks something like:

```c++
namespace MyCustomOpNamespace {

template <typename T>
class ReductionMeanOperator : public uTensor::OperatorInterface<2, 1> {
 public:
  enum names_in : uint8_t { in, axis };
  enum names_out : uint8_t { out };

 protected:
  virtual void compute() {
    //...
  }
};

} // MyCustomOpNamespace
```

### Adding an Operator Eval Snippet

The first thing we will want to do is make sure the generated code maps the correct input and output tensors to their associated names in the runtime op. We start by declaring these names in an `OpEvalSnippet`. From the op description above this looks like "in", "axis", and "out", and can be done with the following code snippet:

```python
class ReductionMeanEvalSnippet(OpEvalSnippet):
    __inputs__ = ["in", "axis"]
    __outputs__ = ["out"]
```

### Writing a Backend Component

The backend component is the meat of the operator code generation process. For the uTensor backend, this can be creating custom constructor parameters, overriding the default declaration in code, and describing the evaluation code for this operator. Once this is done, we just need to register the component in the `OperatorFactory`:

```python
@OperatorFactory.register
class _ReductionMeanOperator(_Operator):
    namespaces = ("MyCustomOpNamespace",)
    op_type = "ReductionMeanOperator"

    # the value returned by this method will be used as
    # the constrcutor parameters as is.
    # In utensor backend, it should return a tuple of string.
    # Since there is no parameters for `MeanOperator`, an empty tuple is returned
    @classmethod
    @must_return_type(Hashable)
    def get_constructor_parameters(cls, op_info):
        return tuple()

    # snippet that calls op's constructor and will be placed in the
    # the initializer list of the model class
    def get_construct_snippet(self, op_var_name):
        return OpConstructSnippet(
            op=self,
            templ_dtypes=[self.in_dtypes[0]],
            op_var_name=op_var_name,
            nested_namespaces=type(self).namespaces,
        )

    # snippet which declares the op
    def get_declare_snippet(self, op_var_name, with_const_params=True):
        return DeclareOpSnippet(
            op=self,
            templ_dtypes=[self.in_dtypes[0]],
            op_var_name=op_var_name,
            nested_namespaces=type(self).namespaces,
            with_const_params=with_const_params,
        )

    # snippet that eval the op
    def get_eval_snippet(self, op_var_name, op_info, tensor_var_map):
        return ReductionMeanEvalSnippet(
            op_info=op_info,
            templ_dtypes=[self.in_dtypes[0]],
            op_name=op_var_name,
            tensor_var_map=tensor_var_map,
            nested_namespaces=type(self).namespaces,
        )

```

#### `get_constructor_parameters`

This method shoule be a `classmethod` which takes one `op_info` (of type `utensor_cgen.ir.base.OperationInfo`) and return a tuple.

All operators in `uTensor` are derieved from `OperatorInterface` which is implemented template class.

Some of them may require parameters for construction. For example, the constructor of `uTensor::TflmSymQuantOps::FullyConnectedOperator` takes one activation parameters (`FullyConnectedOperator(kTfLiteActRelu)` for example). In that case, you need to return a tuple, `('kTfLiteActRelu',)`, in `get_constructor_parameters`. Note that the values in the tuple returned will be used as is. Normally, the tuple will consist of strings.

As for `ReductionMeanOperator` in our example, it should return an empty tuple since it does not require any parameters for construction.

This method is used for deduplication. That is, the `uTensor` backend will generate singlton object for the operator by its namespace, type signature and constructor parameters (values retured by this method). As a result, one should not confused it with `get_construct_snippet` (see below), which is responsible to generate snippet object for operator construction.

#### `get_construct_snippet`

As methioned above, this method is responsible for returning a `Snippet` object which will generate code snippet for operator construction.

The snippet template looks like this:
```cpp
`op_var_name`(`param1`, `param2`, ...)
```
and `op_var_name` and `param*` should be replaced accordingly, operator by operator.

Normally, you can simply return a `OpConstructSnippet` (defined in `utensor_cgen.backend.utensor.snippets.rearch`)

#### `get_declare_snippet`

This method is responsible for returing a `Snippet` which will generate code snippet for operator declaration:
```cpp
`op_type` `op_var_name`;
```
`op_type` (such as `MyNamespace::ReduceMeanOperator`) and `op_var_name` (such as `op1`) will be replaced accordingly, operator by operator.

Normally, returning a `DeclareOpSnippet` (defined in `utensor_cgen.backend.utensor.snippets.rearch`) will be sufficient.

`DeclareOpSnippet` takes following parameters:
- `op`: an `_Operator` instance, noramlly `self`
- `templ_dtypes`: should be a iterable of data types which represent the type signature of the operator
- `op_var_name`: a string of the variable name which holds the reference to the operator in generated code. Normally is given in the `get_declare_snippet` call.
- `nested_namespaces`: a tuple of namespaces. ex: `("MyNamespace", "ReduceFunction")`
- `with_const_params`: it was a hack, always pass `True` for now. This will be removed in the future.

#### `get_eval_snippet`

This method is responsible for returning a `Snippet` which will generate code snippet for operation evalution:
```cpp
`op_var_name`.set_inputs({
    {`input_name`, `in_tensor`}, ....
}).set_outputs({
    {`output_name`, `out_tensor`}, ...
}).eval();
```
`op_var_name`, `input_name`s, `in_tensor`s, `output_name`s and `out_tensor`s should be replaced accordingly, operator by operator.

Since the input names mapping are different accross operators, you need to inherit `OpEvalSnippet` (defined in `utensor_cgen.backend.utensor.snippets.rearch`) and overwrites its `__inputs__` and `__outputs__` class attributes.
`__inputs__` and `__outputs__` should be both list of strings which specify the input/output tensor names respectively.

Normally, an `OpEvalSnippet` will takes following arguments
- `op_info`: an instance of `OperationInfo`, which is used to figure out the input/output tensors required for the operation evaluation
- `templ_dtypes`: should be a iterable of types which defines the type signature of the operator
- `op_name`: a string, is the variable name which holds reference to the operator in the generated code (same as `op_var_name`)
- `tensor_var_map`: is a dictionary which maps tensor name to the variable name which holds reference to the tensor in the generated code

### Lowering an Operator to our Backend Component

After registering the operator backend component in the OperatorFactory, we need to notify the graph lowering engine to make this target available. Right now, this is a simple registry of operator names and namespaces, but will soon be replaced with lowering strategy engine (WIP):

```python
@uTensorRearchGraphLower.CodgenAttributes.register("ReductionMeanOperator")
def handler(op_info):
    op_info.code_gen_attributes["namespaces"] = ("MyCustomOpNamespace",)
```

### Legalizing at the Frontend

It turns out there was a disconnect between the original custom operator naming, `ReductionMeanOperator` and the naming present in the TFLite file `Mean`, so we need to indicate to the Legalizer that these ops need to be renamed:

```python
# legalize all `Mean` to `ReductionMeanOperator`
TFLiteLegalizer.register_op_rename(old_name="Mean", new_name="ReductionMeanOperator")
```

### Registering the new Operator in uTensor CLI

![reduce-model](images/reduceModel.svg)

*Fig: Image of the custom Reduction Mean Model generated in the [Reduction Mean Generation Ipython Notebook](generate_rmean_model.ipynb)*

Our custom op registration is implemented as a normal python module.
We can load as many custom modules as we want with `--plugin` flag in `utensor-cli`:
```bash
# loading module1 and module2 all at once
$ utensor-cli --plugin module1 --plugin module2 ...
```

For example, the following shows how to load our custom operators:
```bash
$ utensor-cli --plugin custom_operators convert tf_models/reduceModel.tflite
```

Now that a model is generated, all that is left to do is add the necessary `#include`s for the target custom runtime operators into the generated model header file.
