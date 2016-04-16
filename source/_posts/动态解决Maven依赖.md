title: 动态解决Maven依赖
date: 2016-04-15 21:27:00
tags:
- Java
- Maven
categories: 博客
---
```java
import java.io.File;
import java.io.FileReader;
import java.io.IOException;
import java.util.Arrays;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;

import org.apache.maven.artifact.repository.ArtifactRepository;
import org.apache.maven.artifact.repository.ArtifactRepositoryPolicy;
import org.apache.maven.artifact.repository.MavenArtifactRepository;
import org.apache.maven.artifact.repository.layout.DefaultRepositoryLayout;
import org.apache.maven.model.Dependency;
import org.apache.maven.model.Model;
import org.apache.maven.model.io.xpp3.MavenXpp3Reader;
import org.apache.maven.project.MavenProject;
import org.codehaus.plexus.util.xml.pull.XmlPullParserException;
import org.sonatype.aether.artifact.Artifact;
import org.sonatype.aether.resolution.DependencyResolutionException;
import org.sonatype.aether.util.artifact.DefaultArtifact;
import org.sonatype.aether.util.artifact.JavaScopes;

import com.jcabi.aether.Aether;

public class MavenUtils {
    private static final String CLSPTH_SEP = File.separator.equals("/") ? ":" : ";";

    /**
     * 加载pom文件,下载依赖,获得classpath
     * @param projectPom        pom文件路径
     * @param localRepoFolder   本地仓库文件夹
     * @return
     * @throws DependencyResolutionException
     * @throws IOException
     * @throws XmlPullParserException
     */
    public static String getClasspathFromMavenProject(File projectPom,
                                                      File localRepoFolder) throws DependencyResolutionException,
            IOException,
            XmlPullParserException {
        MavenProject proj = loadProject(projectPom);

        proj.setRemoteArtifactRepositories(
                Arrays.asList((ArtifactRepository) new MavenArtifactRepository("central",
                        "http://mvn.fdx.net:8080/artifactory/repo", new DefaultRepositoryLayout(),
                        new ArtifactRepositoryPolicy(), new ArtifactRepositoryPolicy())));

        String classpath = "";
        Aether aether = new Aether(proj, localRepoFolder);

        List<Dependency> dependencies = proj.getDependencies();
        Iterator<Dependency> it = dependencies.iterator();

        while (it.hasNext()) {
            org.apache.maven.model.Dependency depend = it.next();

            final Collection<Artifact> deps = aether.resolve(
                    new DefaultArtifact(depend.getGroupId(), depend.getArtifactId(),
                            depend.getClassifier(), depend.getType(), depend.getVersion()),
                    JavaScopes.RUNTIME);

            Iterator<Artifact> artIt = deps.iterator();
            while (artIt.hasNext()) {
                Artifact art = artIt.next();
                classpath = classpath + CLSPTH_SEP + art.getFile().getAbsolutePath();
            }
        }

        return classpath;
    }

    private static MavenProject loadProject(File pomFile) throws IOException,
            XmlPullParserException {
        MavenProject ret = null;
        MavenXpp3Reader mavenReader = new MavenXpp3Reader();

        if (pomFile != null && pomFile.exists()) {
            FileReader reader = null;

            try {
                reader = new FileReader(pomFile);
                Model model = mavenReader.read(reader);
                model.setPomFile(pomFile);

                ret = new MavenProject(model);
            } finally {
                reader.close();
            }
        }
        return ret;
    }

    public static void main(String[] args) throws Exception {
        String classpath = getClasspathFromMavenProject(
                new File("/Users/fdx/pom.xml"),
                new File("/Users/fdx/.m2/repository"));
        System.out.println("classpath = " + classpath);
    }
}
```
```xml
<dependency>
  <groupId>org.apache.maven</groupId>
  <artifactId>maven-core</artifactId>
  <version>3.0.3</version>
</dependency>
<dependency>
  <groupId>com.jcabi</groupId>
  <artifactId>jcabi-aether</artifactId>
  <version>0.7.19</version>
</dependency>
```
