# bajaj
package com.example.bajaj;

import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;

import java.util.*;
import java.util.stream.Collectors;

@SpringBootApplication
public class BajajApp implements CommandLineRunner {

    private final RestTemplate restTemplate = new RestTemplate();
    private final ObjectMapper mapper = new ObjectMapper();

    public static void main(String[] args) {
        SpringApplication.run(BajajApp.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        // Step 1: Prepare request body
        Map<String, String> requestBody = new HashMap<>();
        requestBody.put("name", "Ganapathi Karthik V");
        requestBody.put("regNo", "22BLC1346"); // your RegNo
        requestBody.put("email", "ganapathikarthik.v2022@vitstudent.ac.in");

        // Step 2: Call Generate Webhook API
        String url = "https://bfhldevapigw.healthrx.co.in/hiring/generateWebhook/JAVA";
        ResponseEntity<Map> response = restTemplate.postForEntity(url, requestBody, Map.class);

        if (response.getStatusCode() == HttpStatus.OK && response.getBody() != null) {
            String webhookUrl = (String) response.getBody().get("webhook");
            String accessToken = (String) response.getBody().get("accessToken");

            System.out.println("Webhook: " + webhookUrl);
            System.out.println("Access Token: " + accessToken);

            // Extract the question payload
            Map<String, Object> questionPayload = (Map<String, Object>) response.getBody().get("question");

            // Deserialize payload
            List<User> users = mapper.convertValue(questionPayload.get("users"), new TypeReference<List<User>>() {});
            int findId = (Integer) questionPayload.get("findId");
            int n = (Integer) questionPayload.get("n");

            // Step 3: Solve the Nth level followers problem
            List<Integer> outcome = findNthLevelFollowers(users, findId, n);

            // Step 4: Submit final answer
            submitSolution(webhookUrl, accessToken, "22BLC1346", outcome);

        } else {
            System.out.println("Failed to generate webhook.");
        }
    }

    private List<Integer> findNthLevelFollowers(List<User> users, int findId, int n) {
        // Build graph
        Map<Integer, List<Integer>> graph = users.stream()
                .collect(Collectors.toMap(User::getId, User::getFollows));

        Set<Integer> visited = new HashSet<>();
        Queue<Integer> queue = new LinkedList<>();
        queue.add(findId);
        visited.add(findId);

        int level = 0;
        while (!queue.isEmpty() && level < n) {
            int size = queue.size();
            for (int i = 0; i < size; i++) {
                int current = queue.poll();
                for (int neighbor : graph.getOrDefault(current, List.of())) {
                    if (!visited.contains(neighbor)) {
                        queue.add(neighbor);
                        visited.add(neighbor);
                    }
                }
            }
            level++;
        }

        List<Integer> result = new ArrayList<>(queue);
        Collections.sort(result); // ensure sorted order
        return result;
    }

    private void submitSolution(String webhookUrl, String accessToken, String regNo, List<Integer> outcome) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Authorization", accessToken);

        Map<String, Object> body = new HashMap<>();
        body.put("regNo", regNo);
        body.put("outcome", outcome);

        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(body, headers);

        ResponseEntity<String> response = restTemplate.exchange(
                webhookUrl,
                HttpMethod.POST,
                entity,
                String.class
        );

        System.out.println("Submission Response: " + response.getBody());
    }

    // DTO for User
    public static class User {
        private int id;
        private List<Integer> follows;

        public User(@JsonProperty("id") int id, 
                    @JsonProperty("follows") List<Integer> follows) {
            this.id = id;
            this.follows = follows;
        }

        public int getId() { return id; }
        public List<Integer> getFollows() { return follows; }
    }
}

