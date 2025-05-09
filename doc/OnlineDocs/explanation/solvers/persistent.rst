.. _persistent_solvers:

Persistent Solvers
==================

The purpose of the persistent solver interfaces is to efficiently
notify the solver of incremental changes to a Pyomo model. The
persistent solver interfaces create and store model instances from the
Python API for the corresponding solver. For example, the
:class:`GurobiPersistent<pyomo.solvers.plugins.solvers.gurobi_persistent.GurobiPersistent>`
class maintains a pointer to a gurobipy Model object. Thus, we can
make small changes to the model and notify the solver rather than
recreating the entire model using the solver Python API (or rewriting
an entire model file - e.g., an lp file) every time the model is
solved.

.. warning:: Users are responsible for notifying persistent solver
   interfaces when changes to a model are made!


Using Persistent Solvers
------------------------

The first step in using a persistent solver is to create a Pyomo model
as usual.

>>> import pyomo.environ as pyo
>>> m = pyo.ConcreteModel()
>>> m.x = pyo.Var()
>>> m.y = pyo.Var()
>>> m.obj = pyo.Objective(expr=m.x**2 + m.y**2)
>>> m.c = pyo.Constraint(expr=m.y >= -2*m.x + 5)

You can create an instance of a persistent solver through the SolverFactory.

>>> opt = pyo.SolverFactory('gurobi_persistent')  # doctest: +SKIP

This returns an instance of :py:class:`GurobiPersistent`. Now we need
to tell the solver about our model.

>>> opt.set_instance(m)  # doctest: +SKIP

This will create a gurobipy Model object and include the appropriate
variables and constraints. We can now solve the model.

>>> results = opt.solve()  # doctest: +SKIP

We can also add or remove variables, constraints, blocks, and
objectives. For example,

>>> m.c2 = pyo.Constraint(expr=m.y >= m.x)  # doctest: +SKIP
>>> opt.add_constraint(m.c2)  # doctest: +SKIP

This tells the solver to add one new constraint but otherwise leave
the model unchanged. We can now resolve the model.

>>> results = opt.solve()  # doctest: +SKIP

To remove a component, simply call the corresponding remove method.

>>> opt.remove_constraint(m.c2)  # doctest: +SKIP
>>> del m.c2  # doctest: +SKIP
>>> results = opt.solve()  # doctest: +SKIP

If a pyomo component is replaced with another component with the same
name, the first component must be removed from the solver. Otherwise,
the solver will have multiple components. For example, the following
code will run without error, but the solver will have an extra
constraint. The solver will have both y >= -2*x + 5 and y <= x, which
is not what was intended!

>>> m = pyo.ConcreteModel()  # doctest: +SKIP
>>> m.x = pyo.Var()  # doctest: +SKIP
>>> m.y = pyo.Var()  # doctest: +SKIP
>>> m.c = pyo.Constraint(expr=m.y >= -2*m.x + 5)  # doctest: +SKIP
>>> opt = pyo.SolverFactory('gurobi_persistent')  # doctest: +SKIP
>>> opt.set_instance(m)  # doctest: +SKIP
>>> # WRONG:
>>> del m.c  # doctest: +SKIP
>>> m.c = pyo.Constraint(expr=m.y <= m.x)  # doctest: +SKIP
>>> opt.add_constraint(m.c)  # doctest: +SKIP

The correct way to do this is:

>>> m = pyo.ConcreteModel()  # doctest: +SKIP
>>> m.x = pyo.Var()  # doctest: +SKIP
>>> m.y = pyo.Var()  # doctest: +SKIP
>>> m.c = pyo.Constraint(expr=m.y >= -2*m.x + 5)  # doctest: +SKIP
>>> opt = pyo.SolverFactory('gurobi_persistent')  # doctest: +SKIP
>>> opt.set_instance(m)  # doctest: +SKIP
>>> # Correct:
>>> opt.remove_constraint(m.c)  # doctest: +SKIP
>>> del m.c  # doctest: +SKIP
>>> m.c = pyo.Constraint(expr=m.y <= m.x)  # doctest: +SKIP
>>> opt.add_constraint(m.c)  # doctest: +SKIP

.. warning:: Components removed from a pyomo model must be removed
             from the solver instance by the user.

Additionally, unexpected behavior may result if a component is
modified before being removed.

>>> m = pyo.ConcreteModel()  # doctest: +SKIP
>>> m.b = pyo.Block()  # doctest: +SKIP
>>> m.b.x = pyo.Var()  # doctest: +SKIP
>>> m.b.y = pyo.Var()  # doctest: +SKIP
>>> m.b.c = pyo.Constraint(expr=m.b.y >= -2*m.b.x + 5)  # doctest: +SKIP
>>> opt = pyo.SolverFactory('gurobi_persistent')  # doctest: +SKIP
>>> opt.set_instance(m)  # doctest: +SKIP
>>> m.b.c2 = pyo.Constraint(expr=m.b.y <= m.b.x)  # doctest: +SKIP
>>> # ERROR: The constraint referenced by m.b.c2 does not
>>> # exist in the solver model.
>>> opt.remove_block(m.b)  # doctest: +SKIP 

In most cases, the only way to modify a component is to remove it from
the solver instance, modify it with Pyomo, and then add it back to the
solver instance. The only exception is with variables. Variables may
be modified and then updated with with solver:

>>> m = pyo.ConcreteModel()  # doctest: +SKIP
>>> m.x = pyo.Var()  # doctest: +SKIP
>>> m.y = pyo.Var()  # doctest: +SKIP
>>> m.obj = pyo.Objective(expr=m.x**2 + m.y**2)  # doctest: +SKIP
>>> m.c = pyo.Constraint(expr=m.y >= -2*m.x + 5)  # doctest: +SKIP
>>> opt = pyo.SolverFactory('gurobi_persistent')  # doctest: +SKIP
>>> opt.set_instance(m)  # doctest: +SKIP
>>> m.x.setlb(1.0)  # doctest: +SKIP
>>> opt.update_var(m.x)  # doctest: +SKIP

Working with Indexed Variables and Constraints
----------------------------------------------

The examples above all used simple variables and constraints; in order to use
indexed variables and/or constraints, the code must be slightly adapted:

>>> for v in indexed_var.values():  # doctest: +SKIP
...     opt.add_var(v)
>>> for v in indexed_con.values():  # doctest: +SKIP
...     opt.add_constraint(v)

This must be done when removing variables/constraints, too. Not doing this would
result in AttributeError exceptions, for example:

>>> opt.add_var(indexed_var)          # doctest: +SKIP
>>> # ERROR: AttributeError: 'IndexedVar' object has no attribute 'is_binary'
>>> opt.add_constraint(indexed_con)   # doctest: +SKIP
>>> # ERROR: AttributeError: 'IndexedConstraint' object has no attribute 'body'

The method "is_indexed" can be used to automate the process, for example:

>>> def add_variable(opt, variable):     # doctest: +SKIP
...     if variable.is_indexed():
...         for v in variable.values():
...             opt.add_var(v)
...     else:
...         opt.add_var(v)

Persistent Solver Performance
-----------------------------
In order to get the best performance out of the persistent solvers, use the
"save_results" flag:

>>> import pyomo.environ as pyo
>>> m = pyo.ConcreteModel()
>>> m.x = pyo.Var()
>>> m.y = pyo.Var()
>>> m.obj = pyo.Objective(expr=m.x**2 + m.y**2)
>>> m.c = pyo.Constraint(expr=m.y >= -2*m.x + 5)
>>> opt = pyo.SolverFactory('gurobi_persistent')  # doctest: +SKIP
>>> opt.set_instance(m)  # doctest: +SKIP
>>> results = opt.solve(save_results=False)  # doctest: +SKIP

Note that if the "save_results" flag is set to False, then the following
is not supported.

>>> results = opt.solve(save_results=False, load_solutions=False)  # doctest: +SKIP
>>> if results.solver.termination_condition == TerminationCondition.optimal:
...     m.solutions.load_from(results)  # doctest: +SKIP

However, the following will work:

>>> results = opt.solve(save_results=False, load_solutions=False)  # doctest: +SKIP
>>> if results.solver.termination_condition == TerminationCondition.optimal:
...     opt.load_vars()  # doctest: +SKIP

Additionally, a subset of variable values may be loaded back into the model:

>>> results = opt.solve(save_results=False, load_solutions=False)  # doctest: +SKIP
>>> if results.solver.termination_condition == TerminationCondition.optimal:
...     opt.load_vars(m.x)  # doctest: +SKIP
