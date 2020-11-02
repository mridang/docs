## TIL About: Bean Testing [Advanced]

So we had this issue where we had controllers that ingested JSON, deserialized them (using Jackson) and wrote them to a message queue.

One day  we noticed issues with the data quality in our DBs. We noticed that when the bean was written to the message queue, it was serialzied to JSON (using Jackson). When the consumers read from the queue, they took the JSON payload and deseralied it back into a bean.

This scenario often led us to the issue that beans when deserialised, serialized and deserialised again must yeild the same object. 

This often helped us catch peculiararies in our beans. The test below helps you catch any quirks. Any beans that you'd like tested should simply extend `JacksonBean`.

```java
package com.nosto.jackson;


import org.apache.commons.lang3.builder.EqualsBuilder;
import org.apache.commons.lang3.builder.HashCodeBuilder;
import org.apache.commons.lang3.builder.ToStringBuilder;
import org.apache.commons.lang3.builder.ToStringStyle;

/**
 * A base class for Jackson-mapped beans. All beans should extend this.
 *
 * All sub-classes are automatically unit-tested and they should have
 * a constructor with {@link com.fasterxml.jackson.annotation.JsonCreator}
 * that takes all serialization properties.
 */
public abstract class JacksonBean {

  @Override
    public final boolean equals(Object obj) {
        return EqualsBuilder.reflectionEquals(this, obj);
    }

    @Override
    public final int hashCode() {
        return HashCodeBuilder.reflectionHashCode(this);
    }

    @Override
    public final String toString() {
        return ToStringBuilder.reflectionToString(this, ToStringStyle.MULTI_LINE_STYLE);
    }
}
```



```java
package com.example;

import java.lang.reflect.Modifier;
import java.util.Collection;
import java.util.Optional;
import java.util.Set;
import java.util.stream.Collectors;

import org.junit.runner.RunWith;
import org.junit.runners.Parameterized;
import org.reflections.Reflections;
import org.reflections.util.ClasspathHelper;
import org.reflections.util.ConfigurationBuilder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.BeanDescription;
import com.fasterxml.jackson.databind.JavaType;
import com.fasterxml.jackson.databind.ObjectMapper;

@RunWith(Parameterized.class)
public class JacksonBeanTest extends AbstractJacksonBeanTest<JacksonBean, JacksonBean> {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Logger logger = LoggerFactory.getLogger(JacksonBeanTest.class);
    private static final Classes CLASSES;

    static {
        // when running in IDEA, classloader has tmp/classes/production/playcart/ so we don't need to run play
        Reflections reflections = new Reflections(new ConfigurationBuilder().setUrls(ClasspathHelper.forPackage("com.example")));
        CLASSES = reflections::getSubTypesOf;
    }

    /**
     * @param deserClass Base class for polymorphic deserialization or
     *                   the concrete class for no polymorphic deserialization is used.
     * @param concreteClass Class to be tested
     * @param testName The name of the test case, needed only for a clean {@link java.text.MessageFormat}
     */
    public JacksonBeanTest(Class<? extends JacksonBean> deserClass, Class<? extends JacksonBean> concreteClass, String testName) {
        super(deserClass, concreteClass, objectMapper -> {
            if (deserClass != concreteClass) {
                objectMapper.registerSubtypes(concreteClass);
            }
            return objectMapper;
        });
    }

    @Parameterized.Parameters(name = "{1}")
    public static Collection<Object[]> data() {
        Set<Class<? extends JacksonBean>> jacksonBeans = CLASSES.getSubTypesOf(JacksonBean.class)
                .stream()
                .filter(c -> !Modifier.isAbstract(c.getModifiers()))
                .collect(Collectors.toSet());
        logger.info("Found beans {}", jacksonBeans);
        assertFalse("No beans found", jacksonBeans.isEmpty());
        return jacksonBeans.stream()
                .map(c -> new Object[] {findRootType(c).orElse(c), c, c.getName()})
                .collect(Collectors.toList());
    }

    private static Optional<Class<?>> findRootType(Class<?> clazz) {
        JavaType javaType = MAPPER.getTypeFactory().constructType(clazz);
        BeanDescription beanDescription = MAPPER.getSerializationConfig().introspect(javaType);
        if (beanDescription.getClassAnnotations().get(JsonTypeInfo.class) == null) {
            return Optional.empty();
        } else {
            return findRootType(clazz.getSuperclass()).or(() -> Optional.of(clazz));
        }
    }

    private interface Classes {
        <T> Set<Class<? extends T>> getSubTypesOf(Class<T> type);
    }
}
```



```java
package com.nosto.jackson;

import java.io.IOException;
import java.lang.reflect.Modifier;
import java.time.Instant;
import java.time.OffsetDateTime;
import java.time.ZoneOffset;
import java.util.List;
import java.util.Optional;
import java.util.Random;
import java.util.function.Function;
import java.util.stream.Collectors;
import java.util.stream.IntStream;

import org.jeasy.random.EasyRandom;
import org.jeasy.random.EasyRandomParameters;
import org.junit.Assert;
import org.junit.Test;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.BeanDescription;
import com.fasterxml.jackson.databind.JavaType;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.introspect.AnnotatedField;
import com.fasterxml.jackson.databind.introspect.BeanPropertyDefinition;

public abstract class AbstractJacksonBeanTest<T, U extends T> extends Assert {

    protected static final ObjectMapper MAPPER = new ObjectMapper();
    private static final EasyRandom EASY_RANDOM;

    static {
        Random random = new Random();
        EasyRandomParameters params = new EasyRandomParameters()
                .randomize(OffsetDateTime.class, () -> OffsetDateTime
                        .ofInstant(Instant.ofEpochSecond(random.nextLong() % Instant.now().getEpochSecond()), ZoneOffset.UTC))
                .randomize(java.util.Date.class, () -> new java.util.Date(random.nextLong() % Instant.now().getEpochSecond()));
        EASY_RANDOM = new EasyRandom(params);
    }

    final ObjectMapper mapper;
    final Class<? extends T> deserClass;
    final Class<? extends U> concreteClass;

    public AbstractJacksonBeanTest(Class<? extends U> clazz) {
        this(clazz, clazz, Function.identity());
    }

    public AbstractJacksonBeanTest(Class<? extends T> deserClass,
                                   Class<? extends U> concreteClass,
                                   Function<ObjectMapper, ObjectMapper> mapperInit) {
        this.deserClass = deserClass;
        this.concreteClass = concreteClass;
        this.mapper = mapperInit.apply(MAPPER.copy());
    }

    /**
     * Assert all properties have an equivalent
     * {@link com.fasterxml.jackson.annotation.JsonCreator}
     * parameter
     */
    @Test
    public void constructorParameters() {
        getDescription().findProperties()
                .forEach(property ->
                        assertTrue(String.format("Property %s of %s lacks constructor argument",
                                property.getName(), concreteClass.getName()), property.hasConstructorParameter()));
    }

    /**
     * Generate multiple random objects of the given class
     * and assert serializing and deserializing back returns
     * the original object.
     */
    @Test
    public void serde() {
        IntStream.range(0, 10)
                .forEach(ignored -> {
                    T bean = EASY_RANDOM.nextObject(concreteClass);
                    String json = null;
                    try {
                        json = this.mapper.writerWithDefaultPrettyPrinter().writeValueAsString(bean);
                    } catch (JsonProcessingException e) {
                        throw new RuntimeException("Could not serialize + " + json, e);
                    }
                    try {
                        T fromJson = this.mapper.readValue(json, deserClass);
                        assertEquals(json, bean, fromJson);
                    } catch (IOException e) {
                        throw new RuntimeException("Could not deserialize + " + json, e);
                    }
                });
    }

    /**
     * Check that there is no setters in order to assert that the bean
     * is immutable
     */
    @Test
    public void noSetters() {
        List<BeanPropertyDefinition> properties = getDescription().findProperties()
                .stream()
                .filter(BeanPropertyDefinition::hasSetter)
                .collect(Collectors.toList());
        assertTrue(String.format("Properties should be immutable but the following properties have setters: %s",
                properties
                        .stream()
                        .map(BeanPropertyDefinition::getName)
                        .collect(Collectors.joining(","))), properties.isEmpty());
    }

    /**
     * All fields should be final in order to assert that the bean is
     * immutable.
     */
    @Test
    public void finalProperties() {
        List<AnnotatedField> fields = getDescription().findProperties()
                .stream()
                .flatMap(x -> Optional.ofNullable(x.getField())
                        .stream())
                .filter(x -> !Modifier.isFinal(x.getModifiers()))
                .collect(Collectors.toList());
        assertTrue(String.format("The following fields are not final: %s",
                fields.stream()
                        .map(AnnotatedField::getName)
                        .collect(Collectors.joining(", "))), fields.isEmpty());
    }

    private BeanDescription getDescription() {
        JavaType javaType = mapper.getTypeFactory()
                .constructType(concreteClass);
        return mapper
                .getSerializationConfig()
                .introspect(javaType);
    }
}
```

This suite has been an absolute lifesaver and while not perfect does help to catch all sorts of ser-deser quirks.

All work is credited to Olli Kuonanoja for showing me this approach.
