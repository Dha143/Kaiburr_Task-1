Domain Models

```java
@Document(collection = "tasks")
public class Task {
  @Id private String id;
  private String name, owner, command;
  private List<TaskExecution> taskExecutions = new ArrayList<>();
  // getters, constructors...
}

public class TaskExecution {
  private Instant startTime, endTime;
  private String output;
  // getters, constructors...
}
```

---
2 Repository

```java
public interface TaskRepository extends MongoRepository<Task, String> {
  List<Task> findByNameContainingIgnoreCase(String name);
}
```

---

3. Service + Validation

```java
@Service
public class TaskService {
  @Autowired TaskRepository repo;

  public Task save(Task t) {
    validateCommand(t.getCommand());
    return repo.save(t);
  }

  private void validateCommand(String cmd){
    if (cmd.contains("rm ") || cmd.contains("|") /*...*/) {
      throw new IllegalArgumentException("Unsafe command");
    }
  }

  public TaskExecution runTask(String taskId) throws IOException, InterruptedException {
    Task task = repo.findById(taskId).orElseThrow(() -> new NotFoundException());
    var exec = new TaskExecution();
    exec.setStartTime(Instant.now());
    ProcessBuilder pb = new ProcessBuilder("kubectl", "exec", "<pod>", "--", "sh", "-c", task.getCommand());
    Process p = pb.start();
    String out = new String(p.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
    p.waitFor();
    exec.setEndTime(Instant.now());
    exec.setOutput(out);
    task.getTaskExecutions().add(exec);
    repo.save(task);
    return exec;
  }
}
```

> *Tip:* For Kubernetes execution via API, replace `kubectl exec` with official Java client code. ([stackoverflow.com][2], [github.com][3])

---

4. Controllers & Endpoints

```java
@RestController
@RequestMapping("/tasks")
public class TaskController {
  @Autowired TaskService svc;

  @GetMapping
  public List<Task> getAll(@RequestParam(required = false) String name) {
    return name != null ? svc.findByName(name) : svc.findAll();
  }

  @GetMapping("/{id}")
  public Task getOne(@PathVariable String id) {
    return svc.findById(id).orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
  }

  @PutMapping
  public Task createOrUpdate(@RequestBody Task t) {
    return svc.save(t);
  }

  @DeleteMapping("/{id}")
  public void delete(@PathVariable String id) {
    svc.delete(id);
  }

  @PostMapping("/{id}/run")
  public TaskExecution run(@PathVariable String id) throws Exception {
    return svc.runTask(id);
  }
}
```

---

5. Testing with curl or Postman

| Endpoint           | Example                                                                                                                                     |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Create**         | `curl -X PUT localhost:8080/tasks -H "Content-Type:application/json" -d '{"id":"t1","name":"Echo","owner":"Alice","command":"echo hello"}'` |
| **List all**       | `curl localhost:8080/tasks`                                                                                                                 |
| **Get one**        | `curl localhost:8080/tasks/t1`                                                                                                              |
| **Search by name** | `curl localhost:8080/tasks?name=Echo`                                                                                                       |
| **Run**            | `curl -X POST localhost:8080/tasks/t1/run`                                                                                                  |
| **Delete**         | `curl -X DELETE localhost:8080/tasks/t1`                                                                                                    |
