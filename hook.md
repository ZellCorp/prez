import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.Base64;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class JiraHook {

    // ⚙️ CONFIGURATION JIRA À REMPLIR
    private static final String JIRA_URL = "https://votre-domaine.atlassian.net";
    private static final String JIRA_EMAIL = "votre.email@entreprise.com";
    private static final String JIRA_API_TOKEN = "VOTRE_TOKEN_API_JIRA";

    public static void main(String[] args) {
        try {
            // 1. Récupérer le message du dernier commit
            String commitMsg = executeCommand("git log -1 --pretty=%B");

            // 2. Chercher le footer avec le format exact : jira: #ASJ-479 'commentaire' 'Colonne'
            Pattern pattern = Pattern.compile("(?m)^jira:\\s+#([A-Z0-9]+-\\d+)\\s+'([^']+)'\\s+'([^']+)'\\s*$");
            Matcher matcher = pattern.matcher(commitMsg);

            if (!matcher.find()) {
                // Pas de footer Jira détecté au bon format, on s'arrête silencieusement
                return;
            }

            String issueKey = matcher.group(1).trim();      // ex: ASJ-479
            String comment = matcher.group(2).trim();       // ex: ticket résolu
            String targetColumn = matcher.group(3).trim();  // ex: Terminer

            System.out.println(" [Git Hook] Déplacement de " + issueKey + " vers '" + targetColumn + "'...");

            // 3. Préparer l'authentification (Compatible Java 8)
            String auth = Base64.getEncoder().encodeToString((JIRA_EMAIL + ":" + JIRA_API_TOKEN).getBytes("UTF-8"));

            // 4. Récupérer l'ID de la transition (colonne)
            String transitionId = getTransitionId(auth, issueKey, targetColumn);

            if (transitionId == null) {
                System.out.println(" [Git Hook] Impossible de trouver la transition '" + targetColumn + "' pour le ticket " + issueKey);
                return;
            }

            // 5. Exécuter la transition et ajouter le commentaire (API Jira)
            String payload = "{"
                    + "\"update\": {\"comment\": [{\"add\": {\"body\": \"" + escapeJson(comment) + "\"}}]},"
                    + "\"transition\": {\"id\": \"" + transitionId + "\"}"
                    + "}";

            boolean success = executeTransition(auth, issueKey, payload);

            if (success) {
                System.out.println(" [Git Hook] Succès ! Le ticket " + issueKey + " a été mis à jour.");
            }

        } catch (Exception e) {
            System.out.println(" [Git Hook] Erreur lors de l'exécution : " + e.getMessage());
        }
    }


    private static String getTransitionId(String auth, String issueKey, String targetColumn) throws Exception {
        URL url = new URL(JIRA_URL + "/rest/api/2/issue/" + issueKey + "/transitions");
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestMethod("GET");
        con.setRequestProperty("Authorization", "Basic " + auth);
        con.setRequestProperty("Accept", "application/json");

        if (con.getResponseCode() != 200) {
            return null;
        }

        BufferedReader in = new BufferedReader(new InputStreamReader(con.getInputStream(), "UTF-8"));
        StringBuilder content = new StringBuilder();
        String inputLine;
        while ((inputLine = in.readLine()) != null) {
            content.append(inputLine);
        }
        in.close();
        con.disconnect();

        // Extraction de l'ID via regex
        Pattern p = Pattern.compile("\"id\"\\s*:\\s*\"(\\d+)\"\\s*,\\s*\"name\"\\s*:\\s*\"([^\"]+)\"");
        Matcher m = p.matcher(content.toString());
        while (m.find()) {
            if (m.group(2).trim().equalsIgnoreCase(targetColumn)) {
                return m.group(1);
            }
        }
        return null;
    }

    private static boolean executeTransition(String auth, String issueKey, String payload) throws Exception {
        URL url = new URL(JIRA_URL + "/rest/api/2/issue/" + issueKey + "/transitions");
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestMethod("POST");
        con.setRequestProperty("Authorization", "Basic " + auth);
        con.setRequestProperty("Content-Type", "application/json; utf-8");
        con.setDoOutput(true);

        try (OutputStream os = con.getOutputStream()) {
            byte[] input = payload.getBytes("UTF-8");
            os.write(input, 0, input.length);
        }

        int status = con.getResponseCode();
        con.disconnect();
        
        if (status < 200 || status >= 300) {
            System.out.println(" [Git Hook] Erreur HTTP : " + status);
            return false;
        }
        return true;
    }


    private static String executeCommand(String command) throws Exception {
        Process p = Runtime.getRuntime().exec(command);
        BufferedReader reader = new BufferedReader(new InputStreamReader(p.getInputStream(), "UTF-8"));
        StringBuilder output = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            output.append(line).append("\n");
        }
        p.waitFor();
        return output.toString().trim();
    }

    private static String escapeJson(String text) {
        return text.replace("\"", "\\\"").replace("\n", "\\n");
    }
}
