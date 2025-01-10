# car-tracker-seed
seed for the Full-Stack IOT training project: car-tracker

## directory structure:

**`client`** [*module-client*]: contains ui app (HTML, CSS, JS, fonts, images)      
**`api`** [*module-api*]: contains REST API

## mock sensor: 
[http://mocker.egen.io](http://mocker.egen.io)

## requirements:
[https://learn.egen.io](https://learn.egen.io/courses/overview.html)


package com.example.core.servlets;

import com.google.gson.JsonObject;
import com.google.gson.JsonParser;
import org.apache.commons.io.IOUtils;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.SlingHttpServletResponse;
import org.apache.sling.api.servlets.SlingAllMethodsServlet;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.api.resource.ResourceResolverFactory;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ModifiableValueMap;
import org.osgi.service.component.annotations.Component;
import org.osgi.service.component.annotations.Reference;

import javax.servlet.Servlet;
import javax.servlet.ServletException;
import java.io.IOException;
import java.net.HttpURLConnection;
import java.net.URL;
import java.nio.charset.StandardCharsets;
import java.util.Collections;

@Component(
    service = Servlet.class,
    property = {
        "sling.servlet.paths=/bin/myapi/savejson",
        "sling.servlet.methods=GET"
    }
)
public class ApiToVarServlet extends SlingAllMethodsServlet {

    @Reference
    private ResourceResolverFactory resourceResolverFactory;

    private static final String VAR_FOLDER_PATH = "/var/mydata/api-response";

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws ServletException, IOException {
        // Step 1: Call the external API
        String apiUrl = "https://api.example.com/data"; // Replace with your API endpoint
        HttpURLConnection connection = null;
        String apiResponse;

        try {
            URL url = new URL(apiUrl);
            connection = (HttpURLConnection) url.openConnection();
            connection.setRequestMethod("GET");
            connection.setConnectTimeout(5000);
            connection.setReadTimeout(5000);

            if (connection.getResponseCode() == HttpURLConnection.HTTP_OK) {
                apiResponse = IOUtils.toString(connection.getInputStream(), StandardCharsets.UTF_8);
            } else {
                response.setStatus(SlingHttpServletResponse.SC_BAD_REQUEST);
                response.getWriter().write("Failed to fetch API response.");
                return;
            }
        } finally {
            if (connection != null) {
                connection.disconnect();
            }
        }

        // Step 2: Save the JSON response in the /var folder
        ResourceResolver resourceResolver = null;
        try {
            // Obtain a ResourceResolver with service user
            resourceResolver = resourceResolverFactory.getServiceResourceResolver(
                    Collections.singletonMap(ResourceResolverFactory.SUBSERVICE, "writeService"));

            // Ensure the path exists
            Resource varResource = resourceResolver.getResource(VAR_FOLDER_PATH);
            if (varResource == null) {
                resourceResolver.create(resourceResolver.getResource("/var"), "mydata", Collections.singletonMap("jcr:primaryType", "nt:unstructured"));
                varResource = resourceResolver.create(resourceResolver.getResource("/var/mydata"), "api-response", Collections.singletonMap("jcr:primaryType", "nt:unstructured"));
            }

            // Save the response as a property
            ModifiableValueMap properties = varResource.adaptTo(ModifiableValueMap.class);
            if (properties != null) {
                properties.put("jsonResponse", apiResponse);
                resourceResolver.commit();
            }
        } catch (Exception e) {
            response.setStatus(SlingHttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            response.getWriter().write("Error saving response to /var folder: " + e.getMessage());
            return;
        } finally {
            if (resourceResolver != null && resourceResolver.isLive()) {
                resourceResolver.close();
            }
        }

        // Respond to the client
        response.setStatus(SlingHttpServletResponse.SC_OK);
        response.getWriter().write("API response saved to /var folder successfully.");
    }
}
