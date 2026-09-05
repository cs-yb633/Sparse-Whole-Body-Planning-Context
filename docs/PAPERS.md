# Core Papers

Paper metadata below is based on the locally pinned reference repositories. Any claim not established by those sources is marked **unverified**.

## FlowMP: Learning Motion Fields for Robot Planning with Conditional Flow Matching

- **Authors:** Khang Nguyen et al.
- **Source:** arXiv:2503.06135 (2025), as cited by the local FlowMP repository.
- **Relation to mainline:** Direct methodological basis for conditional, multimodal trajectory generation and for studying acceleration-aware flow fields. The project must adapt the formulation from low-dimensional paths to sparse whole-body anchors conditioned on 3D obstacle geometry.

## ABPolicy: Asynchronous B-spline Flow Policy for Smooth and Responsive Robotic Manipulation

- **Source:** Official local implementation repository; formal publication metadata is **unverified**.
- **Relation to mainline:** Motivates B-spline/control-point representations and asynchronous continuity-aware chunk updates for receding-horizon execution. Its 6-DoF arm setting is not evidence that the same design works for G1 whole-body constraints.

## SanD-Planner: Sample-Efficient Diffusion Planner in B-Spline Space for Robust Local Navigation

- **Source:** arXiv:2602.00923; the pinned local repository states acceptance to RSS 2026.
- **Relation to mainline:** Closest reference for mapping scene observations to compact collision-aware B-spline trajectories and selecting candidates with ESDF costs. The current project generalizes the output from a robot path to sparse trajectories of multiple humanoid body parts and adds motion completion plus physical tracking.

## Reading/verification backlog

- Verify the exact ABPolicy paper identifier and venue from an authoritative source.
- Extract comparable objectives, conditioning variables, output representations, and evaluation metrics from all three papers.
- Add whole-body motion completion and humanoid tracking papers only after documenting their precise role in the current architecture.
